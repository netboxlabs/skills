# Diode Entity Type Catalog

Complete list of entity types supported by the Diode SDK — **diode-sdk-python v1.12.0 / diode-sdk-go v1.9.0**, exposing **104 entity classes** and covering NetBox **4.5.x–4.6.x**. All types are available in both Python and Go SDKs.

> **NetBox 4.6 models:** `CableBundle`, `RackGroup`, and `VirtualMachineType` are new in NetBox 4.6 — ingesting them lands correctly only against a NetBox 4.6 instance with a matching Diode plugin.

## Core Entity Types

These are the most commonly used entities. The **Primary Key** column shows the field used for matching existing objects.

| Entity | Primary Key | Required | Common Optional Fields |
|--------|------------|----------|----------------------|
| **Device** | `name` | `name` | `device_type`, `role`, `site`, `platform`, `manufacturer`, `serial`, `asset_tag`, `status`, `tags`, `tenant`, `location`, `rack`, `cluster`, `custom_fields` |
| **Interface** | `name` + `device` | `name`, `device` | `type`, `enabled`, `mtu`, `speed`, `mode`, `description`, `parent`, `lag`, `bridge`, `untagged_vlan`, `tagged_vlans` |
| **IPAddress** | `address` + `vrf` | `address` (CIDR) | `vrf`, `status`, `role`, `tenant`, `dns_name`, `assigned_object_interface`, `nat_inside`, `tags` |
| **Prefix** | `prefix` + `vrf` | `prefix` | `vrf`, `site`, `status`, `role`, `tenant`, `vlan`, `is_pool`, `tags` |
| **Site** | `name` | `name` | `slug`, `status`, `region`, `group`, `tenant`, `facility`, `time_zone`, `description`, `tags` |
| **DeviceType** | `model` | `model` | `manufacturer`, `slug`, `part_number`, `description`, `tags` |
| **DeviceRole** | `name` | `name` | `slug`, `color`, `description`, `tags` |
| **Manufacturer** | `name` | `name` | `slug`, `description`, `tags` |
| **Platform** | `name` | `name` | `slug`, `manufacturer`, `description`, `tags` |
| **VLAN** | `vid` + `group` | `name` | `vid`, `group`, `status`, `role`, `tenant`, `description`, `tags` |
| **VRF** | `name` | `name` | `rd`, `tenant`, `description`, `tags` |
| **Tenant** | `name` | `name` | `slug`, `group`, `description`, `tags` |
| **Cluster** | `name` | `name` | `type`, `group`, `site`, `status`, `tenant`, `tags` |
| **VirtualMachine** | `name` | `name` | `cluster`, `site`, `role`, `tenant`, `platform`, `status`, `vcpus`, `memory`, `disk`, `tags` |

> **Slug auto-generation:** If `slug` is omitted, the reconciler generates it from the name (lowercased, spaces → hyphens).

## All Entity Types by Category

### DCIM (41 types)

ASN, ASNRange, Cable, **CableBundle** *(4.6)*, CablePath, CableTermination, ConsolePort, ConsoleServerPort, Device, DeviceBay, DeviceConfig, DeviceRole, DeviceType, FrontPort, Interface, InventoryItem, InventoryItemRole, Location, MACAddress, Manufacturer, Module, ModuleBay, ModuleType, ModuleTypeProfile, Platform, PowerFeed, PowerOutlet, PowerPanel, PowerPort, Rack, **RackGroup** *(4.6)*, RackReservation, RackRole, RackType, RearPort, Region, Site, SiteGroup, VirtualChassis, VirtualDeviceContext

### IPAM (15 types)

Aggregate, FHRPGroup, FHRPGroupAssignment, IPAddress, IPRange, Prefix, RIR, Role, RouteTarget, Service, VLAN, VLANGroup, VLANTranslationPolicy, VLANTranslationRule, VRF

### Circuits (11 types)

Circuit, CircuitGroup, CircuitGroupAssignment, CircuitTermination, CircuitType, Provider, ProviderAccount, ProviderNetwork, VirtualCircuit, VirtualCircuitTermination, VirtualCircuitType

### Wireless (3 types)

WirelessLAN, WirelessLANGroup, WirelessLink

### VPN (10 types)

IKEPolicy, IKEProposal, IPSecPolicy, IPSecProfile, IPSecProposal, L2VPN, L2VPNTermination, Tunnel, TunnelGroup, TunnelTermination

### Virtualization (7 types)

Cluster, ClusterGroup, ClusterType, VirtualDisk, VirtualMachine, **VirtualMachineType** *(4.6)*, VMInterface

### Tenancy (6 types)

Contact, ContactAssignment, ContactGroup, ContactRole, Tenant, TenantGroup

### Other (9 types)

CustomField, CustomFieldChoiceSet, CustomLink, GenericObject, JournalEntry, Owner, OwnerGroup, **ScriptModule**, Tag

## String Shorthand Support (Python Only)

Entities with a `PRIMARY_VALUE_MAP` entry support string shorthand — pass a string instead of a full object for nested references:

```python
# String → object mapping examples:
"NYC-DC1"        → Site(name="NYC-DC1")
"Cisco"          → Manufacturer(name="Cisco")
"192.168.1.1/24" → IPAddress(address="192.168.1.1/24")
"Catalyst 9300"  → DeviceType(model="Catalyst 9300")
```

**Types WITHOUT string shorthand** (require full object construction): Aggregate, CablePath, CableTermination, IPRange, and several less common types. When in doubt, use the full object form.

## Go Entity Construction

Go uses struct types with pointer fields. Use helper functions for values:

```go
device := &diode.Device{
    Name:         diode.String("sw-01"),
    DeviceType:   &diode.DeviceType{Model: diode.String("Catalyst 9300")},
    Site:         &diode.Site{Name: diode.String("NYC-DC1")},
    Role:         &diode.DeviceRole{Name: diode.String("Access Switch")},
    Manufacturer: &diode.Manufacturer{Name: diode.String("Cisco")},
    Status:       diode.String("active"),
    Serial:       diode.String("ABC123"),
}
```

Pointer helpers: `diode.String()`, `diode.Int()`, `diode.Int64()`, `diode.Float64()`, `diode.Bool()`
