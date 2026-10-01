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
| **Nested group model** | Recursive hierarchies | Region, Location, DeviceRole | `parent` FK to self; `NestedLtreeGroupModel` (PostgreSQL ltree) on 4.7+, `NestedGroupModel` (MPTT) on 4.5–4.6 |

All three inherit from **NetBoxModel**, which provides: custom fields, tags, export templates, custom links, bookmarks, journaling, change logging, notifications, event rules. Every model you interact with has these features.

> **NetBox 4.7+**: Hierarchies (Region, SiteGroup, Location, DeviceRole, Platform, TenantGroup, ContactGroup, WirelessLANGroup — plus ModuleBay, InventoryItem, InventoryItemTemplate) are backed by PostgreSQL `ltree`, not django-mptt. The `lft`/`rght`/`tree_id`/`level` columns are gone: `level` is a Python property only and **cannot be used in a filter or `order_by`**. Renames/reparents cascade to descendants automatically. The `parent` FK, the REST `parent_id`/`ancestor_id` filters, and descendant-inclusive filters like `?region_id=` are unchanged, so the modeling advice below applies on every version. Plugin/script code: `get_ancestors()`, `get_descendants()`, `get_children()`, `add_related_count()` remain; `get_root()`, `get_family()`, `is_leaf_node()`, `move_to()`, `insert_at()` are gone. (`NestedGroupModel` still exists for plugins but is deprecated.)

> **NetBox 4.5**: DeviceRole and Platform changed from OrganizationalModel to a nested group model — they support parent-child hierarchies on 4.5–4.7.

### App Structure

| App | Scope | Key Models |
|-----|-------|------------|
| **dcim** | Physical infrastructure | Region, Site, Location, Rack, RackType, Device, DeviceType, Manufacturer, DeviceRole, Platform, Interface, MACAddress, PowerPanel/PowerFeed, CoolingSource/CoolingFeed *(4.7)*, ModuleBayType *(4.7)* |
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
- **Services** (on a Device or VM): on 4.7+ use `port_mappings` (`["tcp/53", "udp/53"]`) — one service can span protocols. `protocol` + `ports` are deprecated (removed in 5.0) and read back `null` for multi-protocol services. On 4.5–4.6 use `protocol` + `ports` and create one service per protocol.

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
- **Racks**: assign a **RackType** (Manufacturer → RackType → Rack) and put physical dimensions there. On 4.7 the per-rack `form_factor`, `width`, `outer_width/height/depth/unit` fields are **deprecated**; in 5.0 they are removed and `rack_type` becomes mandatory. `u_height`, `starting_unit`, `desc_units`, `mounting_depth` stay on the Rack.

> **MACAddress** has been a standalone model (multiple per interface, one primary) since NetBox 4.2 — on all of 4.5–4.7 it is not just a field on Interface. **4.7+**: Interface/VMInterface `mac_address` is writable — writing it creates or updates the primary MACAddress in one call; `MACAddress.is_primary` is read-only.

> **NetBox 4.7+** device modeling additions (all optional; ignore on 4.5–4.6):
> - **ModuleBayType** — allow-list of what fits where. M2M `module_bay_types` on ModuleBay, ModuleBayTemplate, and ModuleType. Installation is rejected only when *both* bay and module type declare types and share none; template types propagate to instantiated bays. Read-only `Module.is_bay_compatible` / `ModuleBay.is_module_compatible`.
> - **Module relocation** — PATCH a Module's `module_bay` to move it (device derived from the bay); the module's subtree moves with it. Cross-device moves require no cables, IPs, VLANs, or other topology on the moved components.
> - **Channelized interfaces** — set `channels` (e.g. 4) on the physical parent; create one subinterface per channel with `parent` + `channel_id` (1-based) and type `channel` (or the real transceiver type). One cable terminates on the parent; NetBox traces a path per channel. Same fields on InterfaceTemplate. Don't model breakouts as ad-hoc virtual interfaces.
> - **`end_of_life`** date on DeviceType/ModuleType — use for lifecycle planning instead of a custom field.
> - **Cooling** — `cooling_method` (air/liquid/hybrid/immersion) on Device/DeviceType/ModuleType (device inherits from its type on create); `cooling_capability` (air-only/hybrid/liquid-only) + `cooling_capacity` (kW) on Rack/RackType (rack inherits from type). See §3a for full cooling topology.

### 3a. Power & Cooling (4.7+ for cooling)

```
PowerPanel (site, location?) → PowerFeed (→ rack?) → PowerPort ⇄ PowerOutlet (device components)
CoolingSource (site, location?) → CoolingFeed (→ rack?) → CoolingIntake ← CoolingOutflow (device components)
```

- **CoolingSource** = facility plant (chiller, cooling tower, dry cooler, CRAC/CRAH); `site` required, `location` optional, `type`, `status`, `fluid_type`, `cooling_capacity` (kW)
- **CoolingFeed** = one coolant loop (supply + return) from a source to a `rack` (optional); `status`, `cooling_capacity`, `max_flow`/`max_flow_unit`, `tenant`
- **CoolingIntake** / **CoolingOutflow** = device components (with DeviceType/ModuleType templates). An intake's `cooling_outflow` points to the upstream outflow (usually on another device — a CDU); an outflow's `cooling_intake` points to the intake on the *same* device
- Model CDUs, manifolds, and rear-door heat exchangers as ordinary (zero-U) devices with intake/outflow components — there is no CDU model. Hoses are references, not Cables. The feed serving a device is derived from its rack
- Endpoints: `dcim/cooling-sources/`, `dcim/cooling-feeds/`, `dcim/cooling-intakes/`, `dcim/cooling-outflows/`, plus `-templates/`

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

- **Types:** text, longtext, integer, decimal, boolean, date, datetime, URL, JSON, selection, multi-selection, object, multi-object (13 types — unchanged through 4.7; there is **no** standalone "color" type)
- **4.7+ read format:** selection/multi-selection values come back as `{"value": "prod", "label": "Production"}` objects (REST and GraphQL); still write the raw value. 4.5–4.6 return the raw value
- **4.7+:** URL fields are validated against `ALLOWED_URL_SCHEMES` (scheme-less values become `https://…`); `nulls_first` controls where empty values sort; read-only `status` (`active`/`provisioning`/`deleting`) — create-with-default and delete run as a background job above `BULK_UPDATE_CHUNK_SIZE` objects, and the field is not live until `active`
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
- **Rendered output:** on 4.7+ `config_context` is pre-rendered, cached on the object, and **always** included in Device/VM REST responses (`?exclude=config_context` is silently ignored). On 4.5–4.6 exclude it with `?exclude=config_context` when listing devices/VMs — see [netbox-api-integration](../netbox-api-integration/SKILL.md)

## Dependency Order

When bulk-importing data, create objects in dependency order. Required FKs must exist before the dependent object.

**High-level order:**
1. Organizational models (RIR, Manufacturer, ClusterType, DeviceRole, Platform, IPAM Role, RackRole, ModuleBayType *(4.7)*)
2. Taxonomy hierarchies (Region, SiteGroup, TenantGroup, Tenant) — parents before children
3. Sites → Locations → RackTypes (needs Manufacturer) → Racks → PowerPanels/CoolingSources *(4.7)* → PowerFeeds/CoolingFeeds *(4.7)*
4. DeviceTypes / ModuleTypes (needs Manufacturer; ModuleBayType before ModuleType on 4.7)
5. Devices (needs DeviceType, DeviceRole, Site) → Modules → channel subinterfaces *(4.7)*
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
11. **Racks without a RackType** — Per-rack dimensions are deprecated in 4.7 and gone in 5.0. Define a RackType per physical model and assign it.
12. **Cooling/lifecycle data in custom fields** *(4.7+)* — Use `cooling_method`, `cooling_capability`/`cooling_capacity`, `end_of_life`, and the cooling models instead of `cf_cooling`/`cf_eol`.

## Version Notes

### NetBox 4.7 (2026-09-02) — 4.7+ only; recommend ≥ 4.7.2

| Change | Impact |
|--------|--------|
| **Hierarchies on ltree** | `level` not filterable/orderable; renames cascade; `NestedGroupModel` → `NestedLtreeGroupModel` for plugins. Restores from a 4.7.0 `pg_dump` lose cascade triggers — run `rebuild_ltree_paths --check` (fixed 4.7.1) |
| **Cooling models** | CoolingSource → CoolingFeed → CoolingIntake/CoolingOutflow (+ templates); `cooling_method` on Device/DeviceType/ModuleType; `cooling_capability`/`cooling_capacity` on Rack/RackType |
| **ModuleBayType** | M2M `module_bay_types` on ModuleBay/ModuleBayTemplate/ModuleType; compatibility enforced when both sides declare types; endpoint `dcim/module-bay-types/` |
| **Channelized interfaces** | `channels` on parent, `channel_id` + `parent` on subinterfaces, `channel` interface type; one cable on the parent, one path per channel |
| **Module relocation** | PATCH `module_bay` to move a module (cross-device if no active topology) |
| **Service `port_mappings`** | Replaces `protocol` + `ports` (deprecated, removed 5.0); multi-protocol services |
| **Rack fields deprecated** | `form_factor`, `width`, `outer_*` on Rack — move to RackType; `rack_type` mandatory in 5.0 |
| **`end_of_life`** | Date on DeviceType/ModuleType |
| **Interface `mac_address` writable** | Creates/updates the primary MACAddress; `MACAddress.is_primary` |
| **Custom field selection values** | Returned as `{value, label}`; raw value on write. URL fields validated against `ALLOWED_URL_SCHEMES`; `nulls_first`; read-only `status` with background provisioning |
| **Config context always returned** | Pre-rendered; `?exclude=config_context` ignored |

### NetBox 4.5

| Change | Impact |
|--------|--------|
| DeviceRole → NestedGroupModel | Design role hierarchies (e.g., `network` → `network/router`, `network/switch`) |
| Platform → NestedGroupModel | Design platform hierarchies (e.g., `cisco-ios` → `cisco-ios/xe`, `cisco-ios/xr`) |
| MACAddress standalone model (since 4.2) | MAC addresses are first-class objects, not just interface fields — applies on every supported version |
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
