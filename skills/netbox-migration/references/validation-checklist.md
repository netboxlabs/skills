# Migration Validation Checklist

Run through this checklist after each import phase. All three layers must pass before the migration is considered complete.

## Layer 1: Structural Validation (NetBox accepted it)

### Counts
- [ ] Device count matches source data
- [ ] IP address count matches source data
- [ ] Prefix/subnet count matches source data
- [ ] Site count matches source data
- [ ] VLAN count matches source data
- [ ] VM count matches source data
- [ ] Rack count matches source data

### Required Fields
- [ ] Every device has: name, device_type, role, site
- [ ] Every IP has address in CIDR notation
- [ ] Every prefix has valid CIDR (no host bits set)
- [ ] Every VLAN has vid (1–4094)
- [ ] No import errors logged / no records skipped

### Data Quality
- [ ] No duplicate device names within same site+tenant
- [ ] No duplicate IPs within same VRF
- [ ] Status values are valid NetBox choices (not all defaulting to "active")
- [ ] Manufacturer names are consistent (no "Cisco" / "cisco" / "CISCO" variants)
- [ ] Device type names are consistent per hardware model
- [ ] Slugs are clean (no conflicts, no auto-generated gibberish)

## Layer 2: Relational Validation (connections correct)

### Device Relationships
- [ ] Every device assigned to correct site
- [ ] Every device has correct device type + manufacturer
- [ ] Every device has a role assigned
- [ ] Rack positions match source layout (correct U, correct rack)
- [ ] Tenant assignments match source ownership data

### IP/Interface Relationships
- [ ] Every IP address assigned to correct interface on correct device
- [ ] Primary IP set on devices that should have management IPs
- [ ] No orphaned IPs (assigned IPs with deleted/wrong interfaces)
- [ ] Interface types/speeds match reality where specified

### IPAM Relationships
- [ ] VLAN-to-prefix associations are correct
- [ ] VRF assignments match source routing tables
- [ ] Prefix hierarchy makes sense (parent prefixes contain children)
- [ ] No unexpected overlapping prefixes in same VRF
- [ ] Aggregates cover all address space
- [ ] Prefix utilization looks reasonable (not 0% or 100% unexpectedly)

### Hierarchy
- [ ] Region → Site hierarchy matches geographic reality
- [ ] Site → Location → Rack nesting is correct
- [ ] Location hierarchy represents actual building/floor/room structure

## Layer 3: Operational Validation (matches live network)

**This layer requires Discovery and Assurance.** See [netbox-discovery](../../netbox-discovery/SKILL.md) and [netbox-assurance](../../netbox-assurance/SKILL.md).

### Discovery Checks
- [ ] Network scan finds no IPs missing from NetBox (within managed prefixes)
- [ ] No devices responding on network that aren't in NetBox
- [ ] No devices in NetBox marked active that don't respond on network
- [ ] Discovered interfaces match NetBox interface records
- [ ] Discovered OS/platform versions match NetBox platform assignments

### Assurance Rules (define for your environment)
- [ ] Every active device has a primary IP
- [ ] Every active device has at least one interface with link up
- [ ] Every rack has a site and location
- [ ] Every prefix in active use has correct VRF
- [ ] No stale devices (in NetBox but not seen by discovery in N days)
- [ ] Naming convention compliance (regex check on device names)

## Validation Script Template

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token="...")
errors = []

# --- Layer 1: Counts ---
expected = {"devices": 500, "ips": 2000, "prefixes": 150, "sites": 10}
actual = {
    "devices": nb.dcim.devices.count(),
    "ips": nb.ipam.ip_addresses.count(),
    "prefixes": nb.ipam.prefixes.count(),
    "sites": nb.dcim.sites.count(),
}
for k, v in expected.items():
    if actual[k] < v:
        errors.append(f"Count mismatch: {k} expected>={v}, got {actual[k]}")

# --- Layer 2: Spot checks ---
critical = ["core-rtr-01", "core-rtr-02", "fw-01", "dns-01"]
for name in critical:
    d = nb.dcim.devices.get(name=name)
    if not d:
        errors.append(f"Missing critical device: {name}")
        continue
    if not d.primary_ip:
        errors.append(f"No primary IP: {name}")
    if not d.site:
        errors.append(f"No site: {name}")
    if not d.device_type:
        errors.append(f"No device type: {name}")

# --- Orphaned IPs ---
unassigned = [ip.address for ip in nb.ipam.ip_addresses.filter(assigned_object_id="null")]
if unassigned:
    errors.append(f"{len(unassigned)} unassigned IPs found")

# --- Report ---
if errors:
    print(f"VALIDATION FAILED — {len(errors)} issues:")
    for e in errors:
        print(f"  ✗ {e}")
else:
    print("✓ All validation checks passed")
```

## When to Run Each Layer

| Phase | Layer 1 | Layer 2 | Layer 3 |
|---|---|---|---|
| After sites/racks import | ✓ | ✓ | — |
| After devices import | ✓ | ✓ | — |
| After IPAM import | ✓ | ✓ | — |
| After connections import | ✓ | ✓ | — |
| After all imports complete | ✓ | ✓ | ✓ |
| Post-discovery enrichment | — | ✓ | ✓ |
| Before decommissioning source | ✓ | ✓ | ✓ |
