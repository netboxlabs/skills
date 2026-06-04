---
name: netbox-migration
description: >
  Methodology for migrating data into NetBox from spreadsheets, legacy IPAM/DCIM
  systems, CMDBs, and unstructured sources. Use when planning or executing a data
  migration — covers assessment, cleaning, dependency ordering, import strategy
  selection, validation, and post-migration verification with Discovery and Assurance.
license: Apache-2.0
---

# NetBox Migration

> **Your knowledge of NetBox import mechanisms may be outdated.** Available import methods, bulk operation limits, and Diode entity types evolve between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| NetBox REST API docs | `https://netboxlabs.com/docs/netbox/integrations/rest-api/` | Bulk create/update endpoints |
| Diode docs | `https://netboxlabs.com/docs/diode/` | High-volume ingestion |
| pynetbox repo | `https://github.com/netbox-community/pynetbox` | Python SDK for scripted imports |
| NetBox MCP server | If configured — verify imported data landed correctly | Post-migration validation |

This skill teaches how to **think about and execute** data migrations into NetBox. It covers the full lifecycle: analyzing messy source data, cleaning it, choosing an import strategy, loading it in the right order, and verifying everything landed correctly.

**This is NOT about NetBox version upgrades.** This is about getting data from spreadsheets, legacy tools, or other CMDBs into NetBox for the first time (or re-importing).

## When to Use This Skill

- Given a spreadsheet, CSV, or data dump to import into NetBox
- Migrating from another CMDB/IPAM (phpIPAM, Device42, RackTables, ServiceNow, InfoBlox)
- Planning a NetBox deployment and need to populate initial data
- Consolidating data from multiple sources into NetBox as source of truth

## Quick-Start Decision Tree

**How much data?**
- < 100 objects → NetBox CSV import UI
- 100–10K objects → pynetbox scripts
- 10K+ objects or ongoing sync → Diode SDK

**What's the source?**
- Spreadsheet → Clean first, then pynetbox or CSV import
- Legacy IPAM tool → Check for existing migration tools, else export→transform→import
- Network discovery data → Import basics, then use Discovery to enrich
- Multiple sources → Establish primary source per data type, reconcile conflicts

**One-time or ongoing?**
- One-time → pynetbox scripts with get-or-create pattern
- Ongoing sync → Diode SDK with scheduled runs
- Hybrid → Diode for bulk, pynetbox for relationship fixup

## The Five Phases

Every migration follows this arc. Don't skip phases — the most common failure is jumping straight to import without assessment and cleaning.

### Phase 1: Assessment

**Goal:** Understand what you have, what NetBox needs, and the gap between them.

1. **Inventory your sources** — List every spreadsheet, database, tool, wiki page, and person's head that holds infrastructure data. Identify which source is authoritative for each data type.

2. **Map fields to NetBox models** — For each source field, identify the corresponding NetBox model and field. See [references/source-mapping.md](references/source-mapping.md) for common mappings by source system.

3. **Identify what to keep vs leave behind** — Not everything migrates. NetBox has specific models; data that doesn't fit goes into custom fields, comments, or gets dropped. The principle: **start with what NetBox requires, not what your spreadsheet has.**

4. **Determine scope** — Migrate in phases. Start with the physical hierarchy (sites, racks, devices), then IPAM, then connections, then enrichment. Don't try to do everything at once.

5. **Check for existing tools** — phpIPAM has `ipam-migrator`, RackTables has `racktables2netbox`. Search GitHub before writing custom scripts.

**Tricky mappings to watch for:**
- "Location" in source may mean site, location-within-site, or region in NetBox
- "Device type" in source usually means role, not NetBox's DeviceType (which is hardware model)
- Source has one IP per device → you need to create an Interface, assign the IP, then set it as primary_ip
- "Cisco Catalyst 9300-48P" → split into Manufacturer("Cisco") + DeviceType(model="Catalyst 9300-48P")
- Location hierarchy "Building A, Floor 3, Room 301" → nested Location objects

For NetBox's data model and relationships, see [netbox-data-modeling](../netbox-data-modeling/SKILL.md).

### Phase 2: Data Cleaning

**Principle:** Garbage in, garbage out. Clean BEFORE importing — fixing data in NetBox after import is harder than fixing it in a spreadsheet.

**The top 10 cleaning tasks:**

1. **Normalize names** — Pick a convention (lowercase-hyphenated recommended) and apply it everywhere. "NYC-DC1", "New York DC 1", "nyc dc1" → `nyc-dc1`.

2. **Standardize manufacturers** — "Cisco", "cisco", "Cisco Systems", "CISCO" → one canonical name. Case-sensitivity matters — especially with Diode's reconciler.

3. **Standardize device types** — "C9300-48P", "Catalyst 9300 48P", "WS-C9300-48P" → one model string per hardware model.

4. **Deduplicate** — Same device with different names across sources. Same IP on multiple records. Pick the authoritative record.

5. **Remove stale data** — Devices decommissioned years ago still in spreadsheets. Mark as `decommissioning` or remove.

6. **Validate IPs** — Must be proper CIDR notation. Add `/32` for host addresses. `10.0.0.1/24` is an IP address on a /24, not a prefix — know the difference.

7. **Fill required fields** — Every Device needs: `name`, `device_type`, `role`, `site`. Every IP needs `address` in CIDR. Check NetBox's required fields per model.

8. **Map status values** — Source "production"/"live"/"up" → NetBox `active`. Source "down"/"retired" → NetBox `offline` or `decommissioning`. Valid choices: active, planned, staged, offline, decommissioning, inventory.

9. **Handle multi-value cells** — "10.0.1.1, 10.0.1.2" in one cell → split into separate records.

10. **Remove formatting artifacts** — Excel leading zeros stripped, trailing spaces, BOM characters, merged cells creating blanks, "N/A" and "unknown" values → decide skip or placeholder.

**Community wisdom:** "Fill a few records manually in NetBox, export them, then build your CSV to match that format." NetBox's CSV export headers don't match import headers — always test with a small sample first.

### Phase 3: Import

**Choose your strategy based on the decision framework in [references/import-patterns.md](references/import-patterns.md).**

| Strategy | Best For | Dependency Ordering |
|----------|----------|-------------------|
| CSV Import (UI) | < 1K objects, simple types | Manual — one type at a time |
| pynetbox | 1K–100K, complex logic | Manual — you control the order |
| Diode SDK | 10K+, ongoing sync | Automatic — reconciler handles deps |
| Custom Scripts | Patterned data, in-NetBox transforms | Manual — Django ORM |

**Regardless of strategy, respect dependency order.** Objects must exist before other objects can reference them. See [references/dependency-order.md](references/dependency-order.md) for the complete 11-tier ordering.

**Summary of dependency tiers:**
1. Tenants, tags, manufacturers, RIRs, device roles, rack roles
2. Regions, site groups
3. Sites
4. Locations (within sites)
5. Racks (within locations)
6. Device types + component templates (need manufacturer)
7. Devices (need type, role, site) — components auto-created from templates
8. Interfaces (custom ones beyond templates)
9. VRFs, VLANs, prefixes, IP addresses → assign to interfaces
10. Cables, circuits, providers
11. Virtual machines, clusters, config contexts, custom field values

**The circular dependency problem:** A device's `primary_ip` requires: create device → create interface (auto from template) → create IP → assign IP to interface → update device's `primary_ip`. This requires two passes — first create everything, then set primary IPs.

**Diode eliminates most ordering concerns.** It creates missing dependencies automatically. If you ingest a Device referencing a non-existent Site, Diode creates the Site first. For large migrations, this is a significant advantage. See [netbox-diode](../netbox-diode/SKILL.md).

**Key import rules:**
- **Always use get-or-create patterns** — Makes scripts idempotent and re-runnable
- **Test in staging first** — Use a separate NetBox instance for dry runs
- **Import in phases** — Sites/racks first, verify, then devices, verify, then IPAM
- **Reference objects by name/slug in CSV**, not database IDs
- **DeviceType/ModuleType use YAML format**, not CSV (they include component templates)
- **Bulk create** with pynetbox: `nb.dcim.devices.create([{...}, {...}])` — batch in chunks of ~100 (a recommended size for throughput/memory, **not** a NetBox-enforced limit)
- **Exclude config_context** from API queries during bulk operations (massive performance impact)
- **Reading back large sets for validation:** on NetBox **4.6+** prefer cursor pagination (`?start=<id>&limit=N`) over deep `?offset=` scans — offset slows linearly at high offsets. See [netbox-api-integration](../netbox-api-integration/SKILL.md).

For API patterns, see [netbox-api-integration](../netbox-api-integration/SKILL.md).

### Phase 4: Validation

**Three validation layers — all three are required for a complete migration.**

#### Layer 1: Structural (NetBox accepted it)
- Object counts match source: devices, IPs, prefixes, sites, VLANs
- No import errors or skipped records
- Required fields populated on all objects

#### Layer 2: Relational (connections correct)
- Every device has correct site, device type, manufacturer, and role
- Every IP assigned to the correct interface on the correct device
- Primary IPs set on devices that should have them
- Rack positions match source layout
- VLAN-to-prefix associations correct
- VRF assignments match routing design
- Prefix hierarchy makes sense (containers contain child prefixes)

#### Layer 3: Operational (matches live network)
- Devices in NetBox respond on the network
- IPs in NetBox match actual device configurations
- No devices on network missing from NetBox
- Interface states match reality
- This layer uses **Discovery** and **Assurance** — see Phase 5

See [references/validation-checklist.md](references/validation-checklist.md) for the complete checklist.

**Validation script pattern:**
```python
import pynetbox
nb = pynetbox.api("https://netbox.example.com", token="...")

# Count verification
source_device_count = 500  # from your source data
assert nb.dcim.devices.count() >= source_device_count

# Spot-check critical devices
for name in ["core-rtr-01", "fw-01", "dns-01"]:
    d = nb.dcim.devices.get(name=name)
    assert d, f"Missing: {name}"
    assert d.primary_ip, f"No primary IP: {name}"
    assert d.site, f"No site: {name}"

# Find orphaned IPs
for ip in nb.ipam.ip_addresses.all():
    if not ip.assigned_object:
        print(f"Unassigned: {ip.address}")
```

### Phase 5: Post-Migration

**Migration isn't done when the data is loaded.** It's done when NetBox matches reality and is established as the source of truth.

#### Run Discovery to fill gaps
Spreadsheets never have everything. Use the Orb Agent to discover what's missing:
- **Network scanning (NMAP)** — Find IPs not in NetBox, identify rogue devices
- **Device discovery (NAPALM/SNMP)** — Pull real interface configs, VLANs, MAC addresses, OS versions
- **LLDP/CDP neighbor data** — Discover cable connections automatically

Discovery fills the gaps that source data always has: interfaces, actual IP assignments, MAC addresses, real platform versions, cable topology. See [netbox-discovery](../netbox-discovery/SKILL.md).

#### Run Assurance to verify correctness
Define rules for what "correct" looks like, then check compliance:
- Every active device has a primary IP
- Every active device has at least one interface
- Every rack has a site and location
- Every prefix has a VRF (if multi-VRF environment)
- No duplicate IPs in the same VRF

See [netbox-assurance](../netbox-assurance/SKILL.md).

#### Establish ongoing processes
- Schedule regular Discovery runs to detect drift
- Review Assurance reports weekly
- Establish change process: changes go to NetBox first, then implement on network
- Integrate with automation (Ansible/Terraform pull from NetBox as source of truth)

## Completeness Criteria

A migration is **done** when:

1. ✅ All source data is represented in NetBox (every device, IP, prefix, VLAN)
2. ✅ Relationships are correct (IPs → interfaces → devices → racks → sites)
3. ✅ Hierarchy is structured (regions → sites → locations → racks)
4. ✅ IPAM is organized (prefix hierarchy, VRF scoping, aggregates cover all space)
5. ✅ Naming is consistent (one convention, no duplicates, no ambiguity)
6. ✅ Discovery confirms reality matches (no phantom devices, no missing devices)
7. ✅ Assurance rules pass (compliance checks green)
8. ✅ Source system can be decommissioned (NetBox is now authoritative)

## Anti-Patterns

1. **Export ≠ import format** — NetBox CSV export headers differ from import headers. Always test with a sample.
2. **Case sensitivity** — "Site A" and "site a" create duplicates, especially via Diode.
3. **Interfaces auto-created** — DeviceType templates create interfaces when a device is created. Don't re-create them.
4. **Circular dependency (primary IP)** — Requires two-pass: create device → create IP → assign back.
5. **CSV encoding** — Use UTF-8 without BOM, LF line endings.
6. **Slug conflicts** — Auto-generated slugs may collide. Provide explicit slugs when names are similar.
7. **Device name uniqueness** — Per site+tenant, not globally. Same name in different sites is fine.
8. **Large CSV imports are slow** — For > 1K objects, switch to API or Diode.
9. **Don't migrate everything at once** — Phase it. Sites → devices → IPAM → connections.
10. **"Feeling overwhelmed"** — Normal. Focus on what NetBox has models for. Ignore data that doesn't map.

## References

- [references/source-mapping.md](references/source-mapping.md) — Field mappings by source system
- [references/dependency-order.md](references/dependency-order.md) — Complete 11-tier import order
- [references/validation-checklist.md](references/validation-checklist.md) — Post-import verification checklist
- [references/import-patterns.md](references/import-patterns.md) — Code examples for each import strategy
