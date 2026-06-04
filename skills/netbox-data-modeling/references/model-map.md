# NetBox Model Relationship Map

Complete map of apps, models, base classes, and foreign key relationships. NetBox 4.6 (notes mark anything added since 4.5).

## DCIM

| Model | Base | Required FKs | Optional FKs |
|-------|------|-------------|--------------|
| Region | NestedGroupModel | — | parent (self) |
| SiteGroup | NestedGroupModel | — | parent (self) |
| Site | PrimaryModel | — | region, group→SiteGroup, tenant |
| Location | NestedGroupModel | site | parent (self) |
| RackRole | OrganizationalModel | — | — |
| RackGroup *(4.6)* | OrganizationalModel | — | — |
| Rack | PrimaryModel | site | location, tenant, role→RackRole, group→RackGroup *(4.6)* |
| Manufacturer | OrganizationalModel | — | — |
| DeviceType | PrimaryModel | manufacturer | — |
| ModuleType | PrimaryModel | manufacturer | — |
| DeviceRole | NestedGroupModel | — | parent (self) |
| Platform | NestedGroupModel | — | parent (self), manufacturer |
| Device | PrimaryModel + ConfigContextModel | device_type, role, site | tenant, platform, location, rack, cluster, virtual_chassis, primary_ip4, primary_ip6, oob_ip |
| Module | PrimaryModel | device, module_type | module_bay |
| VirtualChassis | PrimaryModel | — | master→Device |
| VirtualDeviceContext | PrimaryModel | device | tenant |
| MACAddress | PrimaryModel | — | — |
| Interface | ComponentModel | device | — |
| ConsolePort | ComponentModel | device | — |
| PowerPort | ComponentModel | device | — |
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
| Service | PrimaryModel | — | device or VM (generic parent) |

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
| CustomField | ChangeLoggedModel | Typed fields on any model |
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
