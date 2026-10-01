# Dependency Order for NetBox Imports

Objects must be created in dependency order — a Device can't reference a Site that doesn't exist yet. This document lists the complete ordering with API endpoints and required fields.

> **Diode SDK eliminates this concern.** Its reconciler auto-creates dependencies. If you're using Diode, you can skip manual ordering. See [netbox-diode](../../netbox-diode/SKILL.md).

## 11-Tier Import Order

### Tier 1: Organizational & Taxonomic (no dependencies)

| Object | API Endpoint | Required Fields |
|---|---|---|
| TenantGroup | `/api/tenancy/tenant-groups/` | name, slug |
| Tenant | `/api/tenancy/tenants/` | name, slug |
| Tag | `/api/extras/tags/` | name, slug |
| RIR | `/api/ipam/rirs/` | name, slug |
| Manufacturer | `/api/dcim/manufacturers/` | name, slug |
| DeviceRole | `/api/dcim/device-roles/` | name, slug |
| RackRole | `/api/dcim/rack-roles/` | name, slug |
| ContactGroup | `/api/tenancy/contact-groups/` | name, slug |
| ContactRole | `/api/tenancy/contact-roles/` | name, slug |
| Contact | `/api/tenancy/contacts/` | name |

### Tier 2: Geographic Hierarchy

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| Region | `/api/dcim/regions/` | name, slug | parent (self, optional) |
| SiteGroup | `/api/dcim/site-groups/` | name, slug | parent (self, optional) |

### Tier 3: Sites

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| Site | `/api/dcim/sites/` | name, slug | region, group, tenant (all optional) |

### Tier 4: Locations

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| Location | `/api/dcim/locations/` | name, slug, site | site (required), parent (self, optional) |

### Tier 5: Racks

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| RackGroup *(4.6)* | `/api/dcim/rack-groups/` | name, slug | — (flat organizational grouping) |
| Rack | `/api/dcim/racks/` | name, site | site, location, role, **group (RackGroup, 4.6)**, tenant (optional); *(4.7)* `cooling_capability`, `cooling_capacity` (optional) |
| CoolingSource *(4.7)* | `/api/dcim/cooling-sources/` | name, site, type | site (required), location (optional) |
| CoolingFeed *(4.7)* | `/api/dcim/cooling-feeds/` | name, cooling_source | cooling_source (required), rack, tenant (optional) |

### Tier 6: Device Taxonomy

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| ModuleBayType *(4.7)* | `/api/dcim/module-bay-types/` | name, slug | manufacturer (optional) — create before module types / bay templates that reference it |
| DeviceType | `/api/dcim/device-types/` | manufacturer, model, slug | manufacturer; *(4.7)* `end_of_life` (date), `cooling_method` (optional) |
| ModuleType | `/api/dcim/module-types/` | manufacturer, model | manufacturer; *(4.7)* `module_bay_types` (M2M), `end_of_life`, `cooling_method` (optional) |
| Platform | `/api/dcim/platforms/` | name, slug | manufacturer (optional) |

**Note:** DeviceTypes should include component templates (interfaces, power ports, console ports; *(4.7)* cooling intake/outflow templates, `module_bay_types` on module bay templates, `channels`/`channel_id` on interface templates). Use YAML import for DeviceTypes — CSV doesn't support component templates; the YAML importer accepts `end_of_life` and `cooling_method` on 4.7. The [NetBox Device Type Library](https://github.com/netbox-community/devicetype-library) has pre-built definitions.

### Tier 7: Devices

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| Device | `/api/dcim/devices/` | name, device_type, role, site | device_type, role, site, location, rack, platform, tenant (optional) |
| Module | `/api/dcim/modules/` | device, module_bay, module_type | device, module_type; *(4.7)* if both the bay and the module type declare `module_bay_types`, they must share at least one |

Components (interfaces, power ports, console ports; *(4.7)* cooling intakes/outflows) are **auto-created from DeviceType templates**. Don't re-create them — query existing ones and update if needed.

*(4.7)* **Channelized (breakout) interfaces:** set `channels` on the parent interface, then create one subinterface per channel with `type: "channel"`, `parent: <parent id>`, `channel_id: N` (unique per parent). Cables terminate on the parent. *(4.7)* An installed module can be relocated by PATCHing its `module_bay` instead of delete + recreate.

### Tier 8: IPAM Foundation

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| VRF | `/api/ipam/vrfs/` | name | tenant (optional) |
| RouteTarget | `/api/ipam/route-targets/` | name | tenant (optional) |
| Aggregate | `/api/ipam/aggregates/` | prefix, rir | rir |
| IPAM Role | `/api/ipam/roles/` | name, slug | — |
| VLANGroup | `/api/ipam/vlan-groups/` | name, slug | scope (optional) |
| VLAN | `/api/ipam/vlans/` | vid, name | group, role, tenant (optional) |

### Tier 9: IP Space

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| Prefix | `/api/ipam/prefixes/` | prefix | vrf, vlan, role, tenant, site (optional) |
| IPRange | `/api/ipam/ip-ranges/` | start_address, end_address | vrf, role, tenant (optional) |
| IPAddress | `/api/ipam/ip-addresses/` | address | vrf, tenant, assigned_object (optional) |
| Service | `/api/ipam/services/` | name, parent_object_type, parent_object_id, **`port_mappings` *(4.7)*** / `protocol` + `ports` (4.5–4.6) | parent device/VM; ipaddresses (optional) |

After creating IPs and assigning to interfaces, **update Device.primary_ip4/primary_ip6** in a second pass. This is the circular dependency — device must exist to create interface, IP must exist to set primary_ip.

### Tier 10: Connections

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| Provider | `/api/circuits/providers/` | name, slug | — |
| CircuitType | `/api/circuits/circuit-types/` | name, slug | — |
| Circuit | `/api/circuits/circuits/` | cid, provider, type | provider, type |
| CableBundle *(4.6)* | `/api/dcim/cable-bundles/` | name | — (create before cables that reference it) |
| Cable | `/api/dcim/cables/` | a_terminations, b_terminations | endpoints (interfaces, etc.); **bundle (CableBundle, 4.6, optional)** |

### Tier 11: Virtualization & Enrichment

| Object | API Endpoint | Required Fields | Dependencies |
|---|---|---|---|
| ClusterType | `/api/virtualization/cluster-types/` | name, slug | — |
| ClusterGroup | `/api/virtualization/cluster-groups/` | name, slug | — |
| VirtualMachineType *(4.6)* | `/api/virtualization/virtual-machine-types/` | name, slug | — |
| Cluster | `/api/virtualization/clusters/` | name, type | type, group, site (optional) |
| VirtualMachine | `/api/virtualization/virtual-machines/` | name | cluster or site, role, platform, **virtual_machine_type (4.6)**, tenant (optional) |
| VMInterface | `/api/virtualization/interfaces/` | virtual_machine, name | virtual_machine |
| ConfigContext | `/api/extras/config-contexts/` | name, data | — |

## The Primary IP Two-Pass Pattern

```python
# Pass 1: Create device (interfaces auto-created from type template)
device = nb.dcim.devices.create(name="sw-01", device_type=1, role=1, site=1)

# Pass 2: Find the management interface, create IP, assign
mgmt_intf = nb.dcim.interfaces.get(device_id=device.id, name="Management1")
ip = nb.ipam.ip_addresses.create(
    address="10.0.1.1/32",
    assigned_object_type="dcim.interface",
    assigned_object_id=mgmt_intf.id
)

# Pass 3: Set as primary
device.primary_ip4 = ip.id
device.save()
```
