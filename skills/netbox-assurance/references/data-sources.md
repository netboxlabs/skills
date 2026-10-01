# Data Sources

Data sources provide the **actual network state** that Assurance compares against NetBox (the **intended state**). At least one active data source is required for Assurance to produce deviations.

## Source Types

### 1. NetBox Discovery Agents

**Best for:** Broad network scanning, device/interface/IP discovery

The Orb Agent actively polls network devices via SNMP, SSH, and APIs, then feeds collected data into Assurance automatically.

**Setup:**
1. Deploy Orb Agent(s) with network access to target devices
2. Configure agent.yaml with discovery profiles (SNMP communities, credentials, network ranges)
3. Agent data flows automatically into Assurance for comparison

See [netbox-discovery SKILL.md](../../netbox-discovery/SKILL.md) for agent configuration details.

**What Discovery typically collects:**
- Devices (hostname, serial, platform, status)
- Interfaces (name, type, speed, status, MAC address)
- IP addresses and prefixes
- Device types and manufacturers
- Cable/connection topology (varies by protocol)

### 2. Controller Integrations

**Best for:** Managed infrastructure with centralized controllers

Pre-built integrations pull state from infrastructure management platforms:

| Controller | What It Provides |
|-----------|-----------------|
| VMware vCenter | Virtual machines, clusters, networks |
| Cisco Catalyst Center | Devices, interfaces, wireless infrastructure |
| HPE Aruba/Mist | Wireless APs, switches, site hierarchy |

Controller integrations are configured through the NetBox Enterprise management interface.

### 3. Diode SDK (Custom Integrations)

**Best for:** Any data source without a built-in integration

Use the Diode SDK (Python or Go) to programmatically push entity data from any source:

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Interface, Entity

with DiodeClient(
    target="grpc://diode-service:8081",
    app_name="my-custom-source",
    app_version="1.0.0",
) as client:
    # Wrap each object in Entity(); related objects use string shorthand
    # (site="NYC" → Site(name="NYC"), device="switch-01" → Device(name="switch-01"))
    entities = [
        Entity(device=Device(name="switch-01", site="NYC", role="Access Switch")),
        Entity(interface=Interface(name="eth0", device="switch-01", type="1000base-t")),
    ]
    client.ingest(entities=entities)
```

See [netbox-diode SKILL.md](../../netbox-diode/SKILL.md) for complete SDK patterns. Current SDKs: diode-sdk-python 1.14.1, diode-sdk-go 1.12.0 (both with token-fetch backoff on 429/5xx; Go also refreshes the token before expiry) — long-running custom sources should stay on these lines.

## Supported Object Types

Any entity type the ingestion pipeline can represent is eligible for comparison. Common types:

| Object Type | Key Fields Compared |
|------------|-------------------|
| `dcim.device` | Name, serial, platform, status, site, role, device type |
| `dcim.interface` | Name, type, speed, enabled, MAC address, MTU |
| `dcim.site` | Name, status, facility, physical address |
| `dcim.devicetype` | Model, manufacturer, part number |
| `ipam.ipaddress` | Address, status, assigned interface, DNS name |
| `ipam.prefix` | Prefix, status, site, VLAN |

The specific fields compared depend on what the data source provides. If a data source only sends device name and serial number, only those fields are compared.

> The set of comparable object types tracks what the Diode SDK can represent — as the SDK adds entities, those types become eligible for drift detection too. diode-sdk-python 1.14.x / diode-sdk-go 1.12.0 (regenerated for NetBox 4.7.0) add `CoolingSource`, `CoolingFeed`, `CoolingIntake`, `CoolingOutflow` and `ModuleBayType`, plus new fields on existing types (`Interface.channels`/`channel_id`/`mac_address`, `Service.port_mappings`, `DeviceType`/`ModuleType.cooling_method`/`end_of_life`, `Rack.cooling_capability`/`cooling_capacity`). The 4.6 additions were `CableBundle`, `RackGroup`, `VirtualMachineType`. These only compare against a platform running the matching NetBox release. Check the [Diode SDK entity catalog](../../netbox-diode/references/entity-catalog.md#netbox-47-additions) for the authoritative list.
>
> **Service ports:** on NetBox 4.7 a service deviation compares `port_mappings` (`["tcp/53", "udp/53"]`); a data source still sending `protocol` + `ports` describes only single-protocol services and those fields are deprecated (removed in NetBox 5.0). Emit `port_mappings` from new integrations.

## Data Source Selection Guide

| Scenario | Recommended Source |
|----------|-------------------|
| Starting from scratch, need to discover what's on the network | Discovery Agents |
| Already using a controller platform (vCenter, Catalyst Center) | Controller Integration |
| Niche equipment or custom data format | Diode SDK |
| Multiple overlapping sources | All of the above — Assurance handles multiple sources |

## Multiple Data Sources

Assurance supports multiple simultaneous data sources. Each deviation tracks which source produced it, visible in the deviation detail view. When multiple sources report on the same object, the system reconciles them during analysis.

## Troubleshooting Data Sources

| Symptom | Check |
|---------|-------|
| No deviations appearing | Verify source is active and sending data; check connectivity |
| Deviations for wrong object types | Review source configuration and collection scope |
| Stale deviations not clearing | Verify source is still collecting; check scan schedules |
| Too many false positives | Tune source collection scope; verify credentials and access |
