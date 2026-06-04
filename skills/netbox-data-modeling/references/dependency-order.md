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
- IPAM Role
- ContactRole
- TunnelGroup
- WirelessLANGroup (if no parent)

## Tier 1: Hierarchical Taxonomies

Self-referential. Create top-down (parents first).

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

## Tier 3: Sites

- Site (needs: Region optional, SiteGroup optional, Tenant optional)

## Tier 4: Site Children

- Location (needs: Site required, parent Location optional)
- VLANGroup (needs: scope optional — Region/SiteGroup/Site/Location/Rack/ClusterGroup/Cluster, +RackGroup *(4.6)*)

## Tier 5: Racks & VLANs

- Rack (needs: Site required, Location optional, RackRole optional, RackGroup optional *(4.6)*, Tenant optional)
- VLAN (needs: VLANGroup optional, Tenant optional, IPAM Role optional)

## Tier 6: Device Types

- DeviceType (needs: Manufacturer required)
- ModuleType (needs: Manufacturer required)
- Component templates (InterfaceTemplate, etc.) are created on DeviceType

## Tier 7: Clusters & Virtual Infrastructure

- Cluster (needs: ClusterType required, scope optional, Tenant optional)

## Tier 8: Devices

- Device (needs: DeviceType required, DeviceRole required, Site required; Location, Rack, Platform, Tenant, Cluster optional)
- Components (Interface, ConsolePort, PowerPort) auto-created from DeviceType templates
- Module (needs: Device, ModuleType, ModuleBay)

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
- Service (needs: Device or VM)
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
