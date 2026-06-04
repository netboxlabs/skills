---
name: netbox-data-modeling
description: >
  Design and manage NetBox data models effectively. Covers site/location hierarchy,
  IPAM organization, device modeling, tenant assignment, custom fields vs tags vs
  config contexts, dependency ordering, and relationship patterns. Use when planning
  NetBox data structure, importing bulk data, or choosing between extensibility mechanisms.
license: Apache-2.0
---

# NetBox Data Modeling

> **Your knowledge of NetBox data models may be outdated.** Available model types, field options, and relationship patterns evolve between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Data model docs | `https://netboxlabs.com/docs/netbox/models/` | All model types and fields |
| Custom fields docs | `https://netboxlabs.com/docs/netbox/customization/custom-fields/` | Field types, validation, filtering |
| NetBox repo | `https://github.com/netbox-community/netbox` | Model source code, migrations |
| NetBox MCP server | If configured — explore existing data model, inspect object schemas | Discover current structure |

## When to Use This Skill

Load this skill when you need to:
- Design a NetBox data model from scratch or extend an existing one
- Choose between custom fields, tags, and config contexts
- Plan bulk data imports (dependency ordering)
- Understand object relationships and hierarchy patterns
- Model sites, IPAM, devices, or tenancy

For API mechanics (authentication, pagination, error handling), see
[netbox-api-integration](../netbox-api-integration/SKILL.md).

## NetBox Model Architecture

### Base Model Classes

Every NetBox object inherits from one of three base classes:

| Base Class | Purpose | Examples | Key Traits |
|-----------|---------|----------|------------|
| **PrimaryModel** | Real infrastructure objects | Device, Site, Prefix, Rack | Has description, comments, owner |
| **OrganizationalModel** | Categorization/taxonomy | RIR, IPAM Role, ClusterType | Unique name + slug |
| **NestedGroupModel** | Recursive hierarchies | Region, Location, DeviceRole | Parent FK to self (MPTT tree) |

All three inherit from **NetBoxModel**, which provides: custom fields, tags, export templates, custom links, bookmarks, journaling, change logging, notifications, event rules. Every model you interact with has these features.

> **NetBox 4.5**: DeviceRole and Platform changed from OrganizationalModel to **NestedGroupModel** — they now support parent-child hierarchies.

### App Structure

| App | Scope | Key Models |
|-----|-------|------------|
| **dcim** | Physical infrastructure | Region, Site, Location, Rack, Device, DeviceType, Manufacturer, DeviceRole, Platform, Interface, MACAddress |
| **ipam** | IP addressing | RIR, Aggregate, Prefix, IPAddress, IPRange, VRF, VLAN, VLANGroup, ASN |
| **circuits** | Connectivity | Provider, Circuit, CircuitGroup, VirtualCircuit |
| **tenancy** | Ownership | TenantGroup, Tenant, Contact, ContactAssignment |
| **virtualization** | VMs | ClusterType, Cluster, VirtualMachine, VMInterface, VirtualDisk |
| **vpn** | Tunnels/VPNs | Tunnel, TunnelGroup, L2VPN, IKE/IPSec policies |
| **wireless** | Wireless | WirelessLAN, WirelessLANGroup, WirelessLink |

See [references/model-map.md](references/model-map.md) for the complete relationship map.

## Core Design Patterns

### 1. Site & Location Hierarchy

```
Region (geographic, recursive)    SiteGroup (functional, recursive)
         \                              /
          └──────── Site ──────────────┘
                      │
                  Location (recursive, within site)
                      │
                    Rack → Device
```

- **Regions** = geography: continent → country → metro
- **SiteGroups** = function: production, staging, lab, edge
- **Site** = a physical facility; optionally assigned to one Region AND/OR one SiteGroup
- **Locations** = subdivisions within a site: building → floor → room → row
- Use Regions when you need geographic filtering/reporting
- Use SiteGroups when you need functional grouping across geographies
- Both are optional — start simple, add hierarchy when filtering demands it

See [references/model-map.md](references/model-map.md) for the complete relationship map and hierarchy design guidance.

### 2. IPAM Organization

```
RIR → Aggregate (top-level allocation, e.g., 10.0.0.0/8)
         └── Prefix (auto-nests by CIDR containment)
               ├── child Prefixes
               ├── IPRange
               └── IPAddress

VRF ──── scopes Prefixes, IPAddresses, IPRanges
VLAN ←── linked to Prefix (optional)
```

**Key rules:**
- Use `status=container` for summary/aggregate prefixes, `status=active` for allocated
- VRF=null means global routing table. Use VRFs for overlapping address spaces
- Set `enforce_unique=True` on VRFs to prevent duplicate prefixes
- Prefixes auto-nest: creating 10.0.1.0/24 inside existing 10.0.0.0/16 builds the tree automatically

**Scope pattern (4.x):** Prefixes and VLANGroups use **CachedScopeMixin** — a generic FK (`scope_type` + `scope_id`) rather than a direct `site` FK. VLANGroup scope accepts **Region, SiteGroup, Site, Location, Rack, ClusterGroup, Cluster** (this full set since 4.5), plus **RackGroup as of 4.6**.

```python
# Setting scope via API
{"prefix": "10.0.1.0/24", "scope_type": "dcim.site", "scope_id": 42}
```

### 3. Device Modeling

```
Manufacturer → DeviceType (template with component templates)
                              ↓ (instantiation)
DeviceRole + Site + DeviceType → Device (with auto-created components)
```

- **DeviceType** defines the hardware template: interface templates, power ports, module bays
- Creating a Device auto-creates components from its DeviceType's templates
- **Modules** extend devices: module types define additional component templates inserted into module bays
- **DeviceRole** (4.5: hierarchical) — use for config context matching and classification
- **Platform** (4.5: hierarchical) — OS/firmware family; optionally tied to a Manufacturer

> **NetBox 4.5**: MACAddress is now a standalone model, not just a field on Interface.

### 4. Tenant Assignment

Tenant is an **optional FK on nearly every PrimaryModel**: Site, Device, Rack, Prefix, VLAN, VRF, Circuit, VM, Cluster, IPAddress, etc.

**Pattern:** TenantGroup (hierarchy) → Tenant → assign to objects

**Use cases:**
- MSP customer segregation
- Internal department ownership
- Cost center tracking

**Anti-pattern:** Don't overload Tenant for two dimensions (e.g., both "customer" and "department"). Use Tenant for the primary ownership dimension; use custom fields or tags for secondary dimensions.

### 5. Contact Assignment

Contacts use a **generic relation pattern**: ContactAssignment links any object to a Contact with a ContactRole.

```
Contact + ContactRole + any object → ContactAssignment
```

ContactGroups organize contacts hierarchically. Multiple contacts with different roles can be assigned to the same object.

## Extending the Data Model

### Decision: Custom Field vs Tag vs Config Context

| Question | → Custom Field | → Tag | → Config Context |
|----------|---------------|-------|-----------------|
| Does it have a value beyond yes/no? | ✅ | ❌ | ✅ |
| Applied across many object types? | ❌ (scoped) | ✅ | ❌ (devices/VMs only) |
| Need to filter/search by it? | ✅ | ✅ | ❌ (not directly) |
| Used by automation/config rendering? | ❌ | ❌ | ✅ |
| Inherited/computed from hierarchy? | ❌ | ❌ | ✅ |
| Per-object unique value? | ✅ | ❌ | ❌ (matched by criteria) |

**Examples:**
- Warranty expiry date → **custom field** (per-device, typed, filterable)
- PCI-compliant → **tag** (boolean-like, cross-object)
- NTP servers for a site's devices → **config context** (inherited, used in config rendering)

See [references/custom-field-types.md](references/custom-field-types.md) for all field types and decision guidance.

### Custom Fields — Key Points

- **Types:** text, longtext, integer, decimal, boolean, date, datetime, URL, JSON, selection, multi-selection, object, multi-object (13 types — unchanged in 4.6; there is **no** standalone "color" type)
- **JSON fields** accept an optional **`validation_schema`** (4.6+) to enforce a JSON Schema on values
- **Selection/multi-selection** choice sets support per-choice colors (`choice_colors`, 4.6+) — this is what release notes call the "color custom field", not a new field type
- **Object/multi-object fields** create relationships to other NetBox objects — prefer these over storing names in text fields
- **Scope** to specific object types at creation time
- **Group** fields with `group_name` for UI organization
- **Visibility:** always / if-set / hidden
- Filter via API: `?cf_<field_name>=<value>`

### Tags — Key Points

- Properties: name, slug, color, description
- **Restrict** `object_types` to relevant models (don't let every tag appear everywhere)
- Filter via API: `?tag=<slug>`
- No value — presence/absence only. If you need a value, use a custom field

### Config Contexts — Key Points

- JSON data matched to devices/VMs via: regions, site_groups, sites, locations, device_types, roles, platforms, cluster_types, cluster_groups, clusters, tenant_groups, tenants, tags
- **Weight-based merging:** lower weight merges first, higher weight overwrites conflicts
- **Deep merge** for dicts; **replace** for lists
- **Local context data** (on device/VM directly) always wins
- **ConfigContextProfile** enforces JSON Schema validation
- **Performance:** exclude with `?exclude=config_context` when listing devices/VMs

## Dependency Order

When bulk-importing data, create objects in dependency order. Required FKs must exist before the dependent object.

**High-level order:**
1. Organizational models (RIR, Manufacturer, ClusterType, DeviceRole, Platform, IPAM Role, RackRole)
2. Taxonomy hierarchies (Region, SiteGroup, TenantGroup, Tenant)
3. Sites → Locations → Racks
4. DeviceTypes (needs Manufacturer)
5. Devices (needs DeviceType, DeviceRole, Site)
6. IPAM: VRFs → Aggregates → Prefixes → IP Addresses
7. VLANGroups → VLANs
8. Clusters → VMs
9. Circuits (needs Provider, CircuitType)
10. Custom fields, tags, config contexts (can be created at any point but best early)

See [references/dependency-order.md](references/dependency-order.md) for the complete ordered list.

## Anti-Patterns

1. **Flat site structure** — Not using Regions/SiteGroups with 50+ sites. Kills filtering and reporting.
2. **Text custom fields as relationships** — Use object/multi-object custom field types instead of storing names as text.
3. **Duplicate dimensions** — Having both `cf_environment=production` AND tag `production`. Pick one.
4. **Ignoring dependency order** — Creating devices before sites/device types. Scripts fail on missing FKs.
5. **Everything in global VRF** — Model overlapping address spaces properly with VRFs.
6. **Overloading tenant** — Using tenant for two things. One dimension only; use custom fields for the rest.
7. **Giant config contexts** — Store variables, not entire configs. Use config templates for rendering.
8. **Unused roles** — DeviceRole drives config context matching. Design roles deliberately, use 4.5 hierarchy.
9. **Prefix without status** — Always set container/active/reserved. Container = organizational, active = allocated.
10. **Hardcoded PKs** — Use name/slug for lookups. PKs differ across environments.

## Version Notes

### NetBox 4.5

| Change | Impact |
|--------|--------|
| DeviceRole → NestedGroupModel | Design role hierarchies (e.g., `network` → `network/router`, `network/switch`) |
| Platform → NestedGroupModel | Design platform hierarchies (e.g., `cisco-ios` → `cisco-ios/xe`, `cisco-ios/xr`) |
| MACAddress standalone model | MAC addresses are first-class objects, not just interface fields |
| VirtualDeviceContext | Model VDCs on multi-tenant devices |
| CachedScopeMixin on Prefix/VLANGroup | Use `scope_type`/`scope_id` instead of direct `site` FK |
| v2 API tokens | Use `Bearer nbt_<key>.<secret>` format |
| ConfigContextProfile | Validate config context data against JSON Schema |
| VirtualCircuit, CircuitGroup | New circuit modeling options |
| VirtualDisk | Disk modeling for VMs |

### NetBox 4.6 (all 4.6+ only — don't assume on a 4.5.x instance)

| Change | Impact |
|--------|--------|
| **VirtualMachineType** | Reusable VM classification (like DeviceType) supplying default platform/vCPUs/memory; endpoint `virtualization/virtual-machine-types/`. VM gains optional `virtual_machine_type` FK |
| **VM `cluster` now optional** | A VM must be tied to **at least one of** site, cluster, or device — clusterless VMs attached directly to a Device are now first-class |
| **CableBundle** | Logical grouping of cables (conduit/trunk/harness); `Cable.bundle` FK, optional, does not affect tracing; endpoint `dcim/cable-bundles/` |
| **RackGroup (flat)** | Secondary, **non-hierarchical** rack categorization (row/aisle/cage) orthogonal to Location; `Rack.group` FK; endpoint `dcim/rack-groups/`. Can scope VLANGroups |
| **VLANGroup scope += rackgroup** | `rackgroup` added to the VLANGroup scope types (full set: region/sitegroup/site/location/rackgroup/rack/clustergroup/cluster) |
| **ASN `role`** | ASNs can now carry an ipam Role (Roles classify prefixes, VLANs, **and** ASNs) |
| **JSON CF `validation_schema`** | JSON custom fields can enforce a JSON Schema |
| **Choice colors** | Per-choice colors on selection/multiselect choice sets (`choice_colors`) — not a new field type |
| v1 API tokens | **Deprecated in 4.6, removed in 5.0**; v2 `nbt_` tokens return plaintext once at creation (4.6.1) |
