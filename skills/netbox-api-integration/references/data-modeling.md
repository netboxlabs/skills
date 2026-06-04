# Data Modeling

Reference for NetBox data model patterns: dependency order, hierarchies, custom fields, tags, tenants, and natural keys.

## Dependency Order

Objects must be created in dependency order — a child cannot reference a parent that doesn't exist.

```
1. ORGANIZATION (no dependencies)
   ├── Tenant Groups → Tenants
   ├── Regions → Site Groups → Sites → Locations
   └── Contact Groups → Contacts → Contact Assignments

2. DCIM PREREQUISITES
   ├── Manufacturers
   ├── Device Types (requires Manufacturer)
   ├── Module Types (requires Manufacturer)
   ├── Platforms
   ├── Device Roles
   └── Rack Roles

3. RACKS
   └── Racks (requires Site, optional Location)

4. DEVICES
   ├── Devices (requires Device Type, Role, Site)
   ├── Modules (requires Device, Module Type)
   └── Interfaces, Ports (require Device)

5. IPAM
   ├── RIRs
   ├── VRFs (optional Tenant)
   ├── Route Targets
   ├── Aggregates (requires RIR)
   ├── Prefixes (optional VRF, Site, VLAN, Tenant)
   ├── IP Ranges, IP Addresses
   ├── VLAN Groups
   └── VLANs

6. VIRTUALIZATION
   ├── Cluster Types → Cluster Groups → Clusters
   ├── Virtual Machines (require a Cluster OR a Device; NetBox 4.6+ makes `cluster` optional)
   └── VM Interfaces (require VM)

7. CIRCUITS
   ├── Providers → Provider Accounts → Provider Networks
   ├── Circuit Types
   └── Circuits → Circuit Terminations

8. CONNECTIONS (last)
   ├── Cables (require endpoints)
   ├── Wireless Links
   └── Power Paths
```

### Example Population

```python
import pynetbox
nb = pynetbox.api("https://netbox.example.com", token=TOKEN)

# Step 1: Organization
region = nb.dcim.regions.create(name="North America", slug="na")
site = nb.dcim.sites.create(name="NYC-DC1", slug="nyc-dc1", region=region.id, status="active")

# Step 2: Prerequisites
manufacturer = nb.dcim.manufacturers.create(name="Cisco", slug="cisco")
device_type = nb.dcim.device_types.create(manufacturer=manufacturer.id, model="Catalyst 9300", slug="c9300")
role = nb.dcim.device_roles.create(name="Access Switch", slug="access-switch", color="00ff00")

# Step 3: Device (all dependencies exist)
device = nb.dcim.devices.create(name="nyc-dc1-sw01", device_type=device_type.id, role=role.id, site=site.id, status="active")
```

### Idempotent Population

```python
def get_or_create(endpoint, defaults, **lookup):
    existing = endpoint.get(**lookup)
    if existing:
        return existing, False
    return endpoint.create(**{**lookup, **defaults}), True
```

**Alternative:** Use [Diode](../references/diode-integration.md) to skip manual dependency ordering entirely.

## Site Hierarchy

Region and Site Group are **parallel** organizational groupings:

```
Region (geographic)          Site Group (logical)
         \                        /
          └──── Site ────────────┘
                  │
              Location (recursive: Floor → Row → Cage)
                  │
                Rack
                  │
               Device
```

- **Region**: Geographic hierarchy (continent → country → metro)
- **Site Group**: Logical grouping independent of geography (e.g., "Production DCs")
- **Site**: Physical facility
- **Location**: Recursive areas within a site (nested to any depth)

```python
region = nb.dcim.regions.create(name="North America", slug="na")
site_group = nb.dcim.site_groups.create(name="Data Centers", slug="dcs")
site = nb.dcim.sites.create(name="NYC-DC1", slug="nyc-dc1", region=region.id, group=site_group.id, status="active")

floor = nb.dcim.locations.create(name="Floor 1", slug="floor-1", site=site.id)
row = nb.dcim.locations.create(name="Row A", slug="row-a", site=site.id, parent=floor.id)
```

Query by hierarchy: `nb.dcim.devices.filter(region="na")`, `nb.dcim.devices.filter(site_group="dcs")`.

## IPAM Hierarchy

```
RIR → Aggregate → Prefix (hierarchical, can nest) → IP Address
                                                   → IP Range

VRF (scopes Prefixes and IP Addresses)
├── Import/Export Route Targets
└── Associated Prefixes/IPs

VLAN Group (optional) → VLAN → Prefix (many-to-many)
```

```python
rir = nb.ipam.rirs.create(name="ARIN", slug="arin")
aggregate = nb.ipam.aggregates.create(prefix="10.0.0.0/8", rir=rir.id)
parent = nb.ipam.prefixes.create(prefix="10.0.0.0/16", status="container")
child = nb.ipam.prefixes.create(prefix="10.0.1.0/24", status="active")
ip = nb.ipam.ip_addresses.create(address="10.0.1.1/24", status="active")

# Querying
children = nb.ipam.prefixes.filter(within="10.0.0.0/16")
available = nb.ipam.prefixes.get(prefix="10.0.1.0/24").available_ips.list()
```

## Natural Keys

Query by human-readable names instead of numeric IDs:

```python
device = nb.dcim.devices.get(name="switch-01")
site = nb.dcim.sites.get(slug="nyc-dc1")
interface = nb.dcim.interfaces.get(device="switch-01", name="Gi0/1")

devices = nb.dcim.devices.filter(site="nyc-dc1", role="access-switch", status="active")
```

| Object Type | Natural Key |
|-------------|-------------|
| Site | name, slug |
| Device | name |
| Device Type | model + manufacturer |
| Prefix | prefix |
| VLAN | vid + group |

## Custom Fields

Extend the data model for organization-specific needs:

```python
device = nb.dcim.devices.create(
    name="server-01", device_type=1, role=1, site=1,
    custom_fields={"environment": "production", "cost_center": "IT-001", "maintenance_window": "sunday-0200"})

# Update
device.custom_fields["environment"] = "staging"
device.save()

# Filter (cf_ prefix)
production = nb.dcim.devices.filter(cf_environment="production")
```

Custom field types: Text, Integer, Boolean, Date, URL, Selection, Multi-select, Object.

## Tags

Cross-object-type classification:

```python
tag = nb.extras.tags.create(name="PCI-Compliant", slug="pci-compliant", color="ff0000")

device = nb.dcim.devices.get(name="server-01")
device.tags = [{"name": "PCI-Compliant"}]
device.save()

# Query across types
pci_devices = nb.dcim.devices.filter(tag="pci-compliant")
pci_prefixes = nb.ipam.prefixes.filter(tag="pci-compliant")
pci_vlans = nb.ipam.vlans.filter(tag="pci-compliant")
```

Setting `tags=` (or `device.tags = [...]`) **replaces the entire tag set** — unsafe when multiple writers each manage different tags. On **NetBox 4.6+**, use the write-only `add_tags` / `remove_tags` REST fields for concurrency-safe partial edits (see [rest-api-patterns.md](rest-api-patterns.md)).

Tags vs Custom Fields: tags are cross-object and boolean (present/absent); custom fields are type-specific with structured values.

## Tenant Isolation

Logical resource separation for multi-tenant environments:

```python
tenant_group = nb.tenancy.tenant_groups.create(name="Customers", slug="customers")
tenant = nb.tenancy.tenants.create(name="ACME Corp", slug="acme-corp", group=tenant_group.id)

prefix = nb.ipam.prefixes.create(prefix="10.100.0.0/24", tenant=tenant.id, status="active")
device = nb.dcim.devices.create(name="acme-fw01", device_type=1, role=1, site=1, tenant=tenant.id)

# Query by tenant
acme_devices = nb.dcim.devices.filter(tenant="acme-corp")
acme_prefixes = nb.ipam.prefixes.filter(tenant="acme-corp")
```
