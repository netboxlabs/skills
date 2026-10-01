# Source System Field Mapping

Common source systems and how their fields map to NetBox models.

## Universal Field Mappings

| Source Field (common names) | NetBox Model.Field | Notes |
|---|---|---|
| hostname, device_name, host | Device.name | Normalize: lowercase-hyphenated |
| ip, ip_address, mgmt_ip | IPAddress.address | Must include CIDR (`/32` for hosts) |
| site, location, datacenter, dc | Site.name | Source often conflates site + location |
| rack, cabinet | Rack.name | Needs site context |
| rack_unit, u_position | Device.position | Lowest U occupied |
| model, hardware | DeviceType.model | Split from manufacturer |
| manufacturer, vendor, make | Manufacturer.name | Often embedded in model string |
| os, firmware | Platform.name | Normalize version strings |
| role, function | DeviceRole.name | Source "device_type" ≠ NetBox DeviceType |
| serial, serial_number | Device.serial | Often missing or stale |
| vlan, vlan_id | VLAN.vid + VLAN.name | May need to split "100-Management" |
| subnet, network, prefix | Prefix.prefix | Ensure CIDR notation |
| vrf, vrf_name | VRF.name | May include route distinguisher |
| tenant, customer, owner | Tenant.name | Map to tenant model |
| status, state | *.status | Map to: active/planned/staged/offline/decommissioning/inventory |
| description, notes | .description or .comments | comments supports markdown |
| eol, end_of_support, eos_date | DeviceType.end_of_life / ModuleType.end_of_life *(4.7)* | Single `YYYY-MM-DD` date per hardware model ("no longer supported by the manufacturer"). On 4.5–4.6 use a custom field. Multi-date, sourced lifecycle data: [netbox-ndx](../../netbox-ndx/SKILL.md) |
| cooling, cooling_type | DeviceType.cooling_method / Device.cooling_method *(4.7)* | Map to `air`/`liquid`/`hybrid`/`immersion`; Device inherits from DeviceType on create. 4.5–4.6: custom field |
| slot_type, bay_type, cage | ModuleBayType *(4.7)* | Name + slug (optional manufacturer); assign to ModuleType.module_bay_types and module bay templates |
| service, listening_port, protocol | Service.port_mappings *(4.7)* | `["tcp/443", "udp/53"]`; on 4.5–4.6 `protocol` + `ports` (one protocol per service) |

## Source-Specific Guidance

### Spreadsheets (Excel/CSV)
**Most common source.** Expect flat tables with no referential integrity.

Key problems:
- Merged cells → blanks when exported to CSV
- Mixed data types ("DHCP" in IP column, "TBD" in rack column)
- Comments in data fields ("10.0.1.1 (old)", "Rack 5 (moved)")
- Multiple values in one cell ("10.0.1.1, 10.0.1.2")
- Inconsistent naming across tabs/sheets

**Approach:** Export each tab to CSV. Create a mapping document. Clean in a scripting language (Python/pandas), not in Excel.

### phpIPAM
**Data available:** Subnets, IPs, VLANs, VRFs, basic device info.

| phpIPAM | NetBox |
|---|---|
| Sections | → Tenant or custom field (no direct equivalent) |
| Subnets | → Prefix |
| IP Addresses | → IPAddress |
| VLANs | → VLAN |
| VRFs | → VRF |
| Devices (basic) | → Device (needs type/role enrichment) |

**Tools:** `ipam-migrator` (Callum027), `phpipam-netbox-mig`

### RackTables
**Data available:** Racks, devices, IPs, VLANs, cabling, locations.

| RackTables | NetBox |
|---|---|
| Location | → Site or Location |
| Row | → Location (within site) |
| Rack | → Rack |
| Object | → Device |
| IPv4/IPv6 Network | → Prefix |
| IP Address | → IPAddress |
| VLAN | → VLAN |
| Port/Link | → Interface + Cable |

**Tools:** `racktables2netbox` (goebelmeier), `racktables-to-netbox` (bandwidth-intern)

### Device42
**Data available:** Full DCIM + IPAM, circuits, dependencies.

No direct migration tool. Export via Device42 API → transform → import via pynetbox or Diode.

Key mapping: Device42's "device" model is richer than NetBox's — some fields go to custom fields. Hardware-model end-of-life/end-of-support dates map to `DeviceType.end_of_life` on NetBox 4.7+ (custom field on 4.5–4.6).

### ServiceNow CMDB
**Data available:** CIs, relationships, locations, services.

NetBox covers a subset of CMDB scope. Map CI types to NetBox models where they fit; not everything will migrate. Focus on: servers, network devices, locations, IP space.

### Nautobot
**Data available:** Nearly 1:1 with NetBox (forked from it).

Models are very similar. Export via Nautobot API → minor field name adjustments → import via NetBox API. Closest migration path of any source.

### Network Discovery Data (NMAP, SNMP)
**Data available:** IP, MAC, hostname, open ports, OS fingerprint.

**Missing:** Rack position, cable connections, role, tenant. Use as supplementary data after initial migration, or as the starting point if no other source exists.

### Ansible Inventories
**Data available:** Hostnames, roles, platform, some IPs.

Good for device names and grouping (groups → roles or tags). Often incomplete for physical infrastructure.
