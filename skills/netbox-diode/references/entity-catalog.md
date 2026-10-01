# Diode Entity Type Catalog

Complete list of entity types supported by the Diode SDK — **diode-sdk-python v1.14.1 / diode-sdk-go v1.12.0**, exposing **110 entity classes** and covering NetBox **4.5–4.7**. All types are available in both Python and Go SDKs. The generated entity set was last regenerated from NetBox v4.7.0 (python 1.14.0 / go 1.12.0).

> **NetBox 4.7 models:** `CoolingSource`, `CoolingFeed`, `CoolingIntake`, `CoolingOutflow` and `ModuleBayType` are new in NetBox 4.7 — ingesting them (or the new 4.7 fields listed under [NetBox 4.7 Additions](#netbox-47-additions)) lands correctly only against a NetBox 4.7 instance with a matching Diode plugin.
>
> **NetBox 4.6 models:** `CableBundle`, `RackGroup`, and `VirtualMachineType` require NetBox 4.6+.

## Core Entity Types

These are the most commonly used entities. The **Primary Key** column shows the field used for matching existing objects.

| Entity | Primary Key | Required | Common Optional Fields |
|--------|------------|----------|----------------------|
| **Device** | `name` | `name` | `device_type`, `role`, `site`, `platform`, `manufacturer`, `serial`, `asset_tag`, `status`, `tags`, `tenant`, `location`, `rack`, `cluster`, `custom_fields`, `cooling_method` *(4.7)* |
| **Interface** | `name` + `device` | `name`, `device` | `type`, `enabled`, `mtu`, `speed`, `mode`, `description`, `parent`, `lag`, `bridge`, `untagged_vlan`, `tagged_vlans`, `primary_mac_address`, `mac_address` *(4.7)*, `channels` *(4.7)*, `channel_id` *(4.7)* |
| **IPAddress** | `address` + `vrf` | `address` (CIDR) | `vrf`, `status`, `role`, `tenant`, `dns_name`, `assigned_object_interface`, `nat_inside`, `tags` |
| **Prefix** | `prefix` + `vrf` | `prefix` | `vrf`, `site`, `status`, `role`, `tenant`, `vlan`, `is_pool`, `tags` |
| **Site** | `name` | `name` | `slug`, `status`, `region`, `group`, `tenant`, `facility`, `time_zone`, `description`, `tags` |
| **DeviceType** | `model` | `model` | `manufacturer`, `slug`, `part_number`, `description`, `tags`, `cooling_method` *(4.7)*, `end_of_life` *(4.7)* |
| **DeviceRole** | `name` | `name` | `slug`, `color`, `description`, `tags` |
| **Manufacturer** | `name` | `name` | `slug`, `description`, `tags` |
| **Platform** | `name` | `name` | `slug`, `manufacturer`, `description`, `tags` |
| **VLAN** | `vid` + `group` | `name` | `vid`, `group`, `status`, `role`, `tenant`, `description`, `tags` |
| **VRF** | `name` | `name` | `rd`, `tenant`, `description`, `tags` |
| **Tenant** | `name` | `name` | `slug`, `group`, `description`, `tags` |
| **Cluster** | `name` | `name` | `type`, `group`, `status`, `tenant`, `scope_site` / `scope_location` / `scope_region` / `scope_site_group`, `tags` |
| **VirtualMachine** | `name` | `name` | `cluster`, `site`, `role`, `tenant`, `platform`, `status`, `vcpus`, `memory`, `disk`, `virtual_machine_type` *(4.6)*, `tags` |
| **Service** | `name` | `name` | `device` or `virtual_machine`, `port_mappings` *(4.7)*, `protocol` + `ports` *(deprecated on 4.7)*, `ipaddresses`, `description`, `tags` |

> **Slug auto-generation:** If `slug` is omitted, the reconciler generates it from the name (lowercased, spaces → hyphens).

## NetBox 4.7 Additions

Present in the generated code of diode-sdk-python 1.14.0+ and diode-sdk-go 1.12.0+. Go field names are the CamelCase form (`CoolingMethod`, `EndOfLife`, `PortMappings`, `ChannelId`, `ModuleBayTypes`).

### New entity types

| Entity | Primary Key | Notable Fields |
|--------|------------|----------------|
| **CoolingSource** | `name` | `site` or `location`, `type`, `status`, `fluid_type`, `cooling_capacity`; mirrors PowerPanel |
| **CoolingFeed** | `name` | `cooling_source`, `rack`, `status`, `cooling_capacity`, `max_flow`, `max_flow_unit`, `tenant`; mirrors PowerFeed |
| **CoolingIntake** | `name` + `device` | `device` or `module`, `label`, `type`, `diameter`, `diameter_unit`, `max_flow`, `max_flow_unit`, `cooling_outflow` (upstream outflow that serves it) |
| **CoolingOutflow** | `name` + `device` | `device` or `module`, `label`, `type`, `diameter`, `diameter_unit`, `cooling_intake` |
| **ModuleBayType** | `name` | `slug`, `manufacturer`, `color`, `description` |
| **User** | `username` | Minimal type, only used for `RackReservation.user` *(python 1.13+ / go 1.11+)* |

The SDK exposes no `*Template` entities (no `CoolingIntakeTemplate`, `InterfaceTemplate`, `ModuleBayTemplate`) — device-type templates are managed through the REST API.

### New fields on existing entities

| Entity | Field | Type | Notes |
|--------|-------|------|-------|
| `Device` | `cooling_method` | str | `air`, `liquid`, `hybrid`, `immersion` |
| `DeviceType`, `ModuleType` | `cooling_method` | str | as above |
| `DeviceType`, `ModuleType` | `end_of_life` | date (`datetime.datetime` / `*time.Time`) | hardware lifecycle planning |
| `ModuleType`, `ModuleBay` | `module_bay_types` | list of `ModuleBayType` (string shorthand OK) | NetBox validates that a module type and its bay share at least one bay type |
| `Rack`, `RackType` | `cooling_capability`, `cooling_capacity` | str, float | `air-only`, `hybrid`, `liquid-only` |
| `Interface` | `channels`, `channel_id` | int | `channels` on the parent; `channel_id` + `parent` on each `channel`-type subinterface |
| `Interface` | `mac_address` | str | creates/updates the primary MAC in one operation; `primary_mac_address` (a `MACAddress` object) still works |
| `Service` | `port_mappings` | list of str | `["tcp/80", "udp/53"]`; supersedes `protocol` + `ports` |
| `RackReservation` | `user` | `User` (string shorthand = username) | |

```python
from netboxlabs.diode.sdk.ingester import Entity, Interface, Service, CoolingFeed
import datetime

entities = [
    # Channelized breakout: parent declares 4 channels, each child binds by channel_id
    Entity(interface=Interface(device="sw-01", name="Ethernet1", type="100gbase-x-qsfp28", channels=4)),
    Entity(interface=Interface(device="sw-01", name="Ethernet1/1", type="channel", parent="Ethernet1", channel_id=1)),
    # Multi-protocol service
    Entity(service=Service(device="dns-01", name="dns", port_mappings=["tcp/53", "udp/53"])),
    # Cooling loop to a rack
    Entity(cooling_feed=CoolingFeed(name="CDU1-Loop-A", cooling_source="CDU-1", rack="R101", status="active")),
]
```

> **4.5/4.6 targets:** these fields and entities are ignored or rejected by older NetBox/Diode plugin versions. Gate them on the NetBox version you are ingesting into.

## All Entity Types by Category

### DCIM (46 types)

ASN, ASNRange, Cable, **CableBundle** *(4.6)*, CablePath, CableTermination, ConsolePort, ConsoleServerPort, **CoolingFeed** *(4.7)*, **CoolingIntake** *(4.7)*, **CoolingOutflow** *(4.7)*, **CoolingSource** *(4.7)*, Device, DeviceBay, DeviceConfig, DeviceRole, DeviceType, FrontPort, Interface, InventoryItem, InventoryItemRole, Location, MACAddress, Manufacturer, Module, ModuleBay, **ModuleBayType** *(4.7)*, ModuleType, ModuleTypeProfile, Platform, PowerFeed, PowerOutlet, PowerPanel, PowerPort, Rack, **RackGroup** *(4.6)*, RackReservation, RackRole, RackType, RearPort, Region, Site, SiteGroup, VirtualChassis, VirtualDeviceContext

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

### Other (10 types)

CustomField, CustomFieldChoiceSet, CustomLink, GenericObject, JournalEntry, Owner, OwnerGroup, **ScriptModule**, Tag, **User** *(4.7 SDKs; minimal)*

## String Shorthand Support (Python Only)

Entities with a `PRIMARY_VALUE_MAP` entry support string shorthand — pass a string instead of a full object for nested references:

```python
# String → object mapping examples:
"NYC-DC1"        → Site(name="NYC-DC1")
"Cisco"          → Manufacturer(name="Cisco")
"192.168.1.1/24" → IPAddress(address="192.168.1.1/24")
"Catalyst 9300"  → DeviceType(model="Catalyst 9300")
"CDU-1"          → CoolingSource(name="CDU-1")      # 4.7
"SFP28"          → ModuleBayType(name="SFP28")      # 4.7
"jdoe"           → User(username="jdoe")            # RackReservation.user
```

**Types WITHOUT string shorthand** (require full object construction): Aggregate, CablePath, CableTermination, IPRange, and several less common types. When in doubt, use the full object form.

## Go Entity Construction

Go uses struct types with pointer fields. Use helper functions for values:

```go
device := &diode.Device{
    Name:          diode.String("sw-01"),
    DeviceType:    &diode.DeviceType{Model: diode.String("Catalyst 9300")},
    Site:          &diode.Site{Name: diode.String("NYC-DC1")},
    Role:          &diode.DeviceRole{Name: diode.String("Access Switch")},
    Manufacturer:  &diode.Manufacturer{Name: diode.String("Cisco")},
    Status:        diode.String("active"),
    Serial:        diode.String("ABC123"),
    CoolingMethod: diode.String("air"), // 4.7
}
```

Pointer helpers: `diode.String()`, `diode.Bool()`, `diode.Int()`, `diode.Int32()`, `diode.Int64()`, `diode.Uint()`, `diode.Uint32()`, `diode.Uint64()`, `diode.Float32()`, `diode.Float64()`. Date fields (`EndOfLife`) are `*time.Time` — take the address of a `time.Time` value; there is no helper.
