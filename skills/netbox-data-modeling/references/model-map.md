# NetBox Model Relationship Map

Complete map of apps, models, base classes, and foreign key relationships. NetBox 4.7 (notes mark anything added since 4.5; unmarked rows apply to 4.5–4.7).

**Nested group models:** rows marked `NestedGroupModel` are `NestedLtreeGroupModel` (PostgreSQL ltree) on 4.7+ and MPTT-backed `NestedGroupModel` on 4.5–4.6. Same `parent` FK and same REST filters (`parent_id`, `ancestor_id`, descendant-inclusive `region_id`/`site_group_id`/`location_id`) on every version; on 4.7 `level` is not filterable. See [Hierarchical Models](#hierarchical-models-ltree-on-47).

## DCIM

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| Region | NestedGroupModel | — | parent (self) |
| SiteGroup | NestedGroupModel | — | parent (self) |
| Site | PrimaryModel | — | region, group→SiteGroup, tenant |
| Location | NestedGroupModel | site | parent (self) |
| RackRole | OrganizationalModel | — | — |
| RackGroup *(4.6)* | OrganizationalModel | — | — |
| RackType | PrimaryModel | manufacturer, form_factor | — ; *(4.7)* `cooling_capability`, `cooling_capacity` |
| Rack | PrimaryModel | site | location, tenant, role→RackRole, group→RackGroup *(4.6)*, rack_type→RackType (**assign it**: per-rack `form_factor`/`width`/`outer_*` deprecated 4.7, `rack_type` mandatory in 5.0); *(4.7)* `cooling_capability`, `cooling_capacity` (inherited from RackType) |
| PowerPanel | PrimaryModel | site | location |
| PowerFeed | PrimaryModel | power_panel | rack, tenant |
| CoolingSource *(4.7)* | PrimaryModel | site, type | location; `status`, `fluid_type`, `cooling_capacity` (kW) |
| CoolingFeed *(4.7)* | PrimaryModel | cooling_source | rack, tenant; `status`, `cooling_capacity`, `max_flow`/`max_flow_unit` |
| Manufacturer | OrganizationalModel | — | — |
| ModuleBayType *(4.7)* | PrimaryModel | — | manufacturer; `color` |
| DeviceType | PrimaryModel | manufacturer | — ; *(4.7)* `cooling_method`, `end_of_life` |
| ModuleType | PrimaryModel | manufacturer | profile→ModuleTypeProfile; *(4.7)* M2M `module_bay_types`, `cooling_method`, `end_of_life` |
| DeviceRole | NestedGroupModel | — | parent (self) |
| Platform | NestedGroupModel | — | parent (self), manufacturer |
| Device | PrimaryModel + ConfigContextModel | device_type, role, site | tenant, platform, location, rack, cluster, virtual_chassis, primary_ip4, primary_ip6, oob_ip; *(4.7)* `cooling_method` (inherited from DeviceType on create) |
| Module | PrimaryModel | device, module_type, module_bay | — ; *(4.7)* PATCH `module_bay` to relocate; read-only `is_bay_compatible` |
| ModuleBay | ComponentModel (ltree on 4.7) | device | module, parent bay; *(4.7)* M2M `module_bay_types`, read-only `is_module_compatible` |
| VirtualChassis | PrimaryModel | — | master→Device |
| VirtualDeviceContext | PrimaryModel | device | tenant |
| MACAddress | PrimaryModel | — | assigned_object (generic → Interface/VMInterface); *(4.7)* read-only `is_primary` |
| Interface | ComponentModel | device | module, parent, bridge, lag, untagged_vlan, vrf; *(4.7)* `channels` (parent), `channel_id` (subinterface, with `parent`), writable `mac_address` |
| ConsolePort | ComponentModel | device | — |
| PowerPort | ComponentModel | device | module (upstream PowerOutlet/PowerFeed reached via Cable, not FK) |
| PowerOutlet | ComponentModel | device | power_port (same device) |
| CoolingIntake *(4.7)* | ComponentModel | device | module, cooling_outflow (upstream, usually another device) |
| CoolingOutflow *(4.7)* | ComponentModel | device | module, cooling_intake (same device) |
| InventoryItem | ComponentModel (ltree on 4.7) | device | parent (self), manufacturer, role |
| Cable | PrimaryModel | — | bundle→CableBundle *(4.6)* |
| CableBundle *(4.6)* | OrganizationalModel | — | — |

## IPAM

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| RIR | OrganizationalModel | — | — |
| Aggregate | PrimaryModel | rir | tenant |
| Role (IPAM) | OrganizationalModel | — | — |
| VRF | PrimaryModel | — | tenant |
| RouteTarget | PrimaryModel | — | tenant |
| Prefix | PrimaryModel + CachedScopeMixin | — | vrf, tenant, vlan, role, scope (generic→Region/SiteGroup/Site/Location) |
| IPRange | PrimaryModel | start_address, end_address | vrf, tenant, role |
| IPAddress | PrimaryModel | address | vrf, tenant |
| VLANGroup | OrganizationalModel + CachedScopeMixin | — | scope (generic→Region/SiteGroup/Site/Location/Rack/ClusterGroup/Cluster, +RackGroup *(4.6)*) |
| VLAN | PrimaryModel | vid | group→VLANGroup, tenant, role |
| ASN | PrimaryModel | rir, asn | tenant, role→Role(IPAM) *(4.6)* |
| ASNRange | PrimaryModel | rir, start, end | tenant |
| FHRPGroup | PrimaryModel | — | — |
| Service | PrimaryModel | `port_mappings` *(4.7)* / `protocol` + `ports` *(4.5–4.6; deprecated 4.7, removed 5.0)* | device or VM (generic parent), ipaddresses |
| ServiceTemplate | PrimaryModel | `port_mappings` *(4.7)* / `protocol` + `ports` *(4.5–4.6)* | — |

## Circuits

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| Provider | PrimaryModel | — | — |
| ProviderAccount | PrimaryModel | provider | — |
| ProviderNetwork | PrimaryModel | provider | — |
| CircuitType | OrganizationalModel | — | — |
| Circuit | PrimaryModel | provider, type | tenant |
| CircuitGroup | OrganizationalModel | — | — |
| VirtualCircuit | PrimaryModel | provider_network | tenant |

## Tenancy

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| TenantGroup | NestedGroupModel | — | parent (self) |
| Tenant | PrimaryModel | — | group→TenantGroup |
| ContactGroup | NestedGroupModel | — | parent (self) |
| Contact | PrimaryModel | — | group→ContactGroup |
| ContactRole | OrganizationalModel | — | — |
| ContactAssignment | ChangeLoggedModel | contact, role, object (generic) | — |

## Virtualization

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| ClusterType | OrganizationalModel | — | — |
| ClusterGroup | OrganizationalModel | — | — |
| VirtualMachineType *(4.6)* | OrganizationalModel | — | — |
| Cluster | PrimaryModel + CachedScopeMixin | type | scope (generic), tenant |
| VirtualMachine | PrimaryModel + ConfigContextModel | — | cluster *(optional since 4.6)*, site, device, tenant, role, platform, virtual_machine_type *(4.6)* |
| VMInterface | ComponentModel | virtual_machine | — |
| VirtualDisk | ComponentModel | virtual_machine | — |

## VPN

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| TunnelGroup | OrganizationalModel | — | — |
| Tunnel | PrimaryModel | — | group, tenant |
| TunnelTermination | ChangeLoggedModel | tunnel | — |
| L2VPN | PrimaryModel | — | tenant |
| L2VPNTermination | NetBoxModel | l2vpn | — |

## Wireless

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| WirelessLANGroup | NestedGroupModel | — | parent (self) |
| WirelessLAN | PrimaryModel + CachedScopeMixin | — | group, vlan, tenant, scope (generic) |
| WirelessLink | PrimaryModel | interface_a, interface_b | tenant |

## Extras (Customization)

| Model | Base | Purpose |
|-------|------|---------|
| CustomField | ChangeLoggedModel | Typed fields on any model; *(4.7)* `nulls_first`, read-only `status` |
| CustomFieldChoiceSet | ChangeLoggedModel | Selection options for custom fields |
| Tag | ChangeLoggedModel | Cross-object labels |
| ConfigContext | ChangeLoggedModel | JSON data matched to devices/VMs |
| ConfigContextProfile | PrimaryModel | JSON Schema validation for config contexts |

## The CachedScopeMixin Pattern

Used by: **Prefix, VLANGroup, Cluster, WirelessLAN**

Replaces direct `site` FK with a generic foreign key:
- `scope_type` — ContentType (e.g., `dcim.site`, `dcim.region`; `dcim.rackgroup` is valid for VLANGroup as of 4.6)
- `scope_id` — Object PK

The set of allowed `scope_type` values is per-model. VLANGroup accepts the widest set (Region, SiteGroup, Site, Location, Rack, ClusterGroup, Cluster, plus RackGroup in 4.6); Prefix/Cluster/WirelessLAN accept the location-oriented subset.

Cached fields for efficient filtering: `_site`, `_region`, `_location`

```python
# API: set scope on a prefix
{"prefix": "10.0.0.0/24", "scope_type": "dcim.site", "scope_id": 1}

# API: filter by cached fields
GET /api/ipam/prefixes/?site_id=1
```

## Hierarchical Models (ltree on 4.7+)

Region, SiteGroup, Location, DeviceRole, Platform, TenantGroup, ContactGroup, WirelessLANGroup, ModuleBay, InventoryItem, InventoryItemTemplate.

| Concern | 4.5–4.6 (django-mptt) | 4.7+ (PostgreSQL `ltree`) |
|---------|----------------------|---------------------------|
| Parent link | `parent` FK | `parent` FK (unchanged) |
| Depth | `level` column — filterable | `level` Python property — **not** filterable/orderable |
| Rename/reparent | Descendant ordering stale until rebuild | Cascades via DB triggers |
| REST filters | `parent_id`, `ancestor_id`, descendant-inclusive `region_id`/`site_group_id`/`location_id` | Same |
| ORM helpers | Full MPTT API | Only `get_ancestors()`, `get_descendants()`, `get_children()`, `add_related_count()` |
| Plugin base class | `NestedGroupModel` | `NestedLtreeGroupModel` (`NestedGroupModel` deprecated) |

Design rule (all versions): keep hierarchies 2–3 levels deep, name siblings uniquely under each parent, create parents before children in imports. On 4.7, an instance restored from a 4.7.0 `pg_dump` may have stale paths — `manage.py rebuild_ltree_paths --check` reports affected models.
