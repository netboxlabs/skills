# Data Import Dependency Order

Create objects in this order for bulk imports. Each tier depends only on tiers above it.

## Tier 0: Independent Organizational Models

No dependencies. Create first.

- RIR
- Manufacturer
- ClusterType
- ClusterGroup
- VirtualMachineType *(4.6)*
- CircuitType
- RackRole
- RackGroup *(4.6)* — flat organizational grouping
- CableBundle *(4.6)* — group cables before creating them (Tier 13)
- ModuleBayType *(4.7)* — optional `manufacturer`; assign to ModuleTypes (Tier 6) and module bay templates
- IPAM Role
- ContactRole
- TunnelGroup
- WirelessLANGroup (if no parent)

## Tier 1: Hierarchical Taxonomies

Self-referential. Create top-down (parents first). On 4.7+ these are ltree-backed: same `parent` FK, but a child's `name` must be unique under its parent and renames cascade to descendants automatically.

- Region (parent → child)
- SiteGroup (parent → child)
- TenantGroup (parent → child)
- ContactGroup (parent → child)
- DeviceRole (parent → child) — **4.5: now hierarchical**
- Platform (parent → child) — **4.5: now hierarchical**
- WirelessLANGroup (nested levels)

## Tier 2: Core Organizational Objects

- Tenant (needs: TenantGroup optional)
- Contact (needs: ContactGroup optional)
- Provider
- ProviderAccount (needs: Provider)
- ProviderNetwork (needs: Provider)
- RackType (needs: Manufacturer required, `form_factor` required) — define one per physical rack model; Racks (Tier 5) should reference it

## Tier 3: Sites

- Site (needs: Region optional, SiteGroup optional, Tenant optional)

## Tier 4: Site Children

- Location (needs: Site required, parent Location optional)
- VLANGroup (needs: scope optional — Region/SiteGroup/Site/Location/Rack/ClusterGroup/Cluster, +RackGroup *(4.6)*)
- PowerPanel (needs: Site required, Location optional)
- CoolingSource *(4.7)* (needs: Site required, `type` required; Location optional)

## Tier 5: Racks & VLANs

- Rack (needs: Site required, Location optional, RackType optional — **assign it**; per-rack dimensions deprecated 4.7, `rack_type` mandatory 5.0 —, RackRole optional, RackGroup optional *(4.6)*, Tenant optional)
- PowerFeed (needs: PowerPanel required, Rack optional)
- CoolingFeed *(4.7)* (needs: CoolingSource required, Rack optional, Tenant optional; rack must be in the source's site)
- VLAN (needs: VLANGroup optional, Tenant optional, IPAM Role optional)

## Tier 6: Device Types

- DeviceType (needs: Manufacturer required; *(4.7)* optional `cooling_method`, `end_of_life`)
- ModuleType (needs: Manufacturer required; *(4.7)* optional M2M `module_bay_types`, `cooling_method`, `end_of_life`)
- Component templates (InterfaceTemplate, ModuleBayTemplate, CoolingIntakeTemplate/CoolingOutflowTemplate *(4.7)*, etc.) are created on DeviceType/ModuleType. *(4.7)* ModuleBayTemplate `module_bay_types` propagate to instantiated bays; InterfaceTemplate supports `channels`/`channel_id`/`parent`

## Tier 7: Clusters & Virtual Infrastructure

- Cluster (needs: ClusterType required, scope optional, Tenant optional)

## Tier 8: Devices

- Device (needs: DeviceType required, DeviceRole required, Site required; Location, Rack, Platform, Tenant, Cluster optional)
- Components (Interface, ConsolePort, PowerPort, CoolingIntake/CoolingOutflow *(4.7)*) auto-created from DeviceType templates
- Module (needs: Device, ModuleType, ModuleBay; *(4.7)* if both bay and module type declare `module_bay_types` they must share one)
- Channel subinterfaces *(4.7)* (needs: parent Interface with `channels` set; one subinterface per `channel_id`, 1-based)
- CoolingIntake → `cooling_outflow` links *(4.7)* (needs: the upstream CoolingOutflow's device — usually a CDU — to exist first)

## Tier 9: Virtual Machines

- VirtualMachine (needs: Cluster optional *(4.6: now optional — was required)*, Site optional, DeviceRole optional, Platform optional, VirtualMachineType optional *(4.6)*, Tenant optional)
- VMInterface (needs: VirtualMachine)
- VirtualDisk (needs: VirtualMachine)

## Tier 10: IPAM

- VRF (needs: Tenant optional)
- RouteTarget (needs: Tenant optional)
- Aggregate (needs: RIR required, Tenant optional)
- Prefix (needs: VRF optional, VLAN optional, IPAM Role optional, Tenant optional, scope optional)
- IPRange (needs: VRF optional, Tenant optional, IPAM Role optional)
- IPAddress (needs: VRF optional, Tenant optional)
- ASN / ASNRange (needs: RIR required, Tenant optional; ASN adds IPAM Role optional *(4.6)*)

## Tier 11: IP Assignments

- Interface → IP address assignments
- Device primary_ip4 / primary_ip6 (needs: IPAddress assigned to device interface)
- Service (needs: Device or VM; `port_mappings` on 4.7+, `protocol` + `ports` on 4.5–4.6)
- FHRPGroup + FHRPGroupAssignment

## Tier 12: Circuits

- Circuit (needs: Provider, CircuitType, Tenant optional)
- CircuitTermination (needs: Circuit, Site/ProviderNetwork)
- CircuitGroup, CircuitGroupAssignment
- VirtualCircuit (needs: ProviderNetwork)

## Tier 13: Connections & Links

- Cable (needs: two endpoints — interfaces, ports, etc.; CableBundle optional *(4.6)*)
- WirelessLink (needs: two Interfaces)
- Tunnel, TunnelTermination
- L2VPN, L2VPNTermination

## Tier 14: Customization & Metadata

Can be created at any time, but best established early:

- CustomField + CustomFieldChoiceSet (before importing data that uses them)
- Tag (before importing data that references them)
- ConfigContext (after the objects it matches exist)
- ContactAssignment (after both contacts and target objects exist)

## Notes

- **Order within a tier** doesn't matter
- **Optional FKs** can be set later via PATCH if needed
- **Custom field data** is set on the object itself, not as a separate call
- When scripting imports, validate each tier completes before starting the next
