# Import Strategy Patterns

Code examples and decision framework for each import strategy.

## Decision Matrix

| Factor | CSV Import | pynetbox | Diode SDK | Custom Scripts |
|---|---|---|---|---|
| Dependency ordering | Manual | Manual | **Automatic** | Manual (ORM) |
| Throughput | Low | Medium | **High** | High (ORM) |
| Idempotency | By ID only | Get-or-create | **Built-in upsert** | Manual |
| Error handling | UI feedback | Programmatic | Response errors | Django exceptions |
| Best volume | < 1K | 1K–100K | **10K+** | Any |
| External data access | CSV only | Any | Any | File upload only |
| Code required | None | Python | Python | Python + Django |
| Relationship handling | Single type | **Full control** | Auto-resolved | **Full control** |

## Strategy 1: CSV Import (NetBox UI)

**Best for:** Quick imports under 1K objects, non-technical users.

**How to learn the format:** Create 2–3 objects manually in NetBox, then export as CSV. Use that export as your template — but note that export headers may differ from import headers.

**Key rules:**
- One object type per import
- Reference related objects by name/slug, not database ID
- Headers must exactly match the import form fields
- UTF-8 encoding, no BOM, LF line endings
- DeviceType and ModuleType use **YAML format**, not CSV

**Gotcha:** NetBox's CSV export format ≠ import format. The export includes extra columns and different header names. Always verify against the import form.

## Strategy 2: pynetbox (REST API)

**Best for:** Medium-large imports with complex logic and relationship handling.

### Basic get-or-create pattern
```python
import pynetbox
nb = pynetbox.api("https://netbox.example.com", token="nbt_abc123.xxxxxxxx")

def get_or_create(app_model, search_params, create_params):
    """Idempotent object creation."""
    obj = app_model.get(**search_params)
    if obj:
        return obj, False
    return app_model.create(**{**search_params, **create_params}), True

# Example: create a site
site, created = get_or_create(
    nb.dcim.sites,
    {"name": "nyc-dc1"},
    {"slug": "nyc-dc1", "status": "active", "region": {"name": "US-East"}}
)
```

### Bulk import from CSV
```python
import csv
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token="nbt_abc123.xxxxxxxx")

# Phase 1: Sites
with open("sites.csv") as f:
    sites = []
    for row in csv.DictReader(f):
        if not nb.dcim.sites.get(name=row["name"]):
            sites.append({"name": row["name"], "slug": row["slug"], "status": "active"})
    if sites:
        nb.dcim.sites.create(sites)  # bulk create; ~100 per call is a good chunk size, not a NetBox cap

# Phase 2: Devices (sites must exist first)
with open("devices.csv") as f:
    devices = []
    for row in csv.DictReader(f):
        if not nb.dcim.devices.get(name=row["name"], site=row["site"]):
            devices.append({
                "name": row["name"],
                "device_type": {"model": row["model"]},
                "role": {"name": row["role"]},
                "site": {"name": row["site"]},
                "status": row.get("status", "active"),
            })
    # Bulk create in batches (~100 is a sensible chunk size — not a NetBox limit; tune for your data)
    for i in range(0, len(devices), 100):
        nb.dcim.devices.create(devices[i:i+100])

# Phase 3: Set primary IPs (devices + IPs must exist)
with open("devices.csv") as f:
    for row in csv.DictReader(f):
        if not row.get("mgmt_ip"):
            continue
        device = nb.dcim.devices.get(name=row["name"], site=row["site"])
        intf = nb.dcim.interfaces.get(device_id=device.id, name="Management1")
        if not intf:  # fallback to first interface
            intfs = list(nb.dcim.interfaces.filter(device_id=device.id))
            intf = intfs[0] if intfs else None
        if intf:
            ip = nb.ipam.ip_addresses.create(
                address=row["mgmt_ip"],
                assigned_object_type="dcim.interface",
                assigned_object_id=intf.id,
            )
            device.primary_ip4 = ip.id
            device.save()
```

**Performance:** Use `?exclude=config_context` on device queries, `?brief=True` for lookups, batch bulk creates (~100/request — a tuning choice, not a NetBox limit), cache site/type lookups. On NetBox **4.6+** use cursor pagination (`?start=`) instead of deep `?offset=` when reading back large sets for validation, and add `?fields=` to project only the columns you need.

**Provenance tagging (NetBox 4.6+):** to stamp imported objects with an "imported-from-X" tag without clobbering existing tags, use the write-only `add_tags` / `remove_tags` serializer fields instead of read-modify-writing the full `tags` list — safer for re-runnable importers and concurrent writers.

## Strategy 3: Diode SDK

**Best for:** Large imports, ongoing sync, minimal ordering concerns.

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import (
    Device, DeviceType, Interface, IPAddress, Site, Manufacturer,
    DeviceRole, Platform, Prefix,
)

with DiodeClient(
    target="grpc://localhost:8081",
    app_name="migration-script",
    app_version="1.0",
) as client:
    entities = []

    # No need to pre-create Site, Manufacturer, DeviceType, Role —
    # Diode creates them automatically if they don't exist
    for row in source_data:
        entities.append(Device(
            name=row["hostname"],
            device_type=DeviceType(
                model=row["model"],
                manufacturer=Manufacturer(name=row["vendor"]),
            ),
            role=DeviceRole(name=row["role"]),
            site=Site(name=row["site"]),
            platform=Platform(name=row["os"]),
            serial=row.get("serial", ""),
            status="active",
        ))

    # Ingest in chunks (Diode handles ordering)
    for i in range(0, len(entities), 500):
        resp = client.ingest(entities=entities[i:i+500])
        if resp.errors:
            for err in resp.errors:
                print(f"Error: {err.message}")
```

**Advantages:** No dependency ordering, upsert semantics, slug auto-generation, high throughput.

**Limitations:** Write-only (use REST API to verify), case-sensitive matching, less attribute control, requires Diode server.

For full Diode patterns, see [netbox-diode](../../netbox-diode/SKILL.md).

## Strategy 4: Custom Scripts (In-NetBox)

**Best for:** Patterned data generation, transforms on existing NetBox data.

```python
from dcim.models import Site
from ipam.models import VLAN, VLANGroup
from extras.scripts import Script

class StandardVLANs(Script):
    class Meta:
        name = "Generate Standard VLANs"

    VLANS = [(10, "Management"), (20, "Servers"), (30, "Users"), (99, "Native")]

    def run(self, data, commit):
        for site in Site.objects.all():
            group, _ = VLANGroup.objects.get_or_create(
                name=f"{site.name}-vlans", scope=site,
                defaults={"slug": f"{site.slug}-vlans"})
            for vid, name in self.VLANS:
                _, created = VLAN.objects.get_or_create(
                    vid=vid, group=group, defaults={"name": name, "status": "active"})
                if created:
                    self.log_success(f"Created VLAN {vid} at {site.name}")
```

For custom script development patterns, see [netbox-custom-scripts](../../netbox-custom-scripts/SKILL.md).

## Recommended Strategy by Scenario

| Scenario | Primary Strategy | Secondary |
|---|---|---|
| Small org, spreadsheet | CSV import | pynetbox for relationships |
| Large org, from CMDB | Diode SDK | pynetbox for fixup |
| Ongoing sync from external | Diode SDK | — |
| One-time with custom logic | pynetbox | — |
| Patterned data (standard VLANs) | Custom Scripts | — |
| Mixed sources, phased | Diode for bulk + pynetbox for edges | — |
