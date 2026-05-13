# Diode Integration

Reference for using [Diode](https://github.com/netboxlabs/diode) for high-volume data ingestion into NetBox.

## What is Diode?

Diode is a data ingestion service from NetBox Labs that:
- **Resolves dependencies automatically** — specify objects by name, not ID
- **Creates missing objects** — referenced objects created if they don't exist
- **Eliminates ordering concerns** — no need to create objects in dependency order
- **Uses gRPC protocol** — high-performance transport
- **Supports 70+ entity types** — all major NetBox object types

## When to Use Diode vs Direct API

| Scenario | Use |
|----------|-----|
| Network discovery pushing data | **Diode** |
| Bulk data migrations | **Diode** |
| Scripts creating many related objects | **Diode** |
| Reading/querying NetBox data | REST/GraphQL |
| Single object CRUD | REST API |
| Complex filtered searches | REST/GraphQL |

## Prerequisites

- NetBox 4.2.3+
- Diode Server deployed
- Diode NetBox Plugin installed
- `pip install netboxlabs-diode-sdk`

## Basic Usage

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Entity

with DiodeClient(
    target="grpc://diode.example.com:8080/diode",
    app_name="my-discovery-tool",
    app_version="1.0.0",
) as client:
    # Specify dependencies by NAME — no IDs needed!
    device = Device(
        name="switch-nyc-01",
        device_type="Cisco Catalyst 9300",  # Created if missing
        manufacturer="Cisco",                # Created if missing
        site="NYC-DC1",                      # Created if missing
        role="Access Switch",                # Created if missing
        serial="ABC123456",
        status="active",
        tags=["production", "network"],
    )
    response = client.ingest([Entity(device=device)])
    if response.errors:
        print(f"Errors: {response.errors}")
```

> **Uniqueness:** Name-based references resolve per model constraints. Devices are unique by name + site. Interfaces by name + device. Check model docs for uniqueness constraints.

## Before and After Comparison

**Without Diode (manual dependency management):**
```python
mfr = nb.dcim.manufacturers.create(name="Cisco", slug="cisco")
dt = nb.dcim.device_types.create(manufacturer=mfr.id, model="C9300", slug="c9300")
site = nb.dcim.sites.create(name="NYC-DC1", slug="nyc-dc1")
role = nb.dcim.device_roles.create(name="Access", slug="access")
device = nb.dcim.devices.create(name="switch-01", device_type=dt.id, site=site.id, role=role.id)
```

**With Diode:**
```python
device = Device(name="switch-01", device_type="C9300", manufacturer="Cisco",
                site="NYC-DC1", role="Access")
client.ingest([Entity(device=device)])
```

## Bulk Ingestion

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Interface, Entity

discovered = [
    {"name": "sw-01", "type": "Catalyst 9300", "site": "NYC", "interfaces": ["Gi0/1", "Gi0/2"]},
    {"name": "sw-02", "type": "Catalyst 9300", "site": "NYC", "interfaces": ["Gi0/1", "Gi0/2"]},
]

with DiodeClient(target="grpc://diode.example.com:8080/diode",
                 app_name="network-scanner", app_version="2.0.0") as client:
    entities = []
    for dev in discovered:
        entities.append(Entity(device=Device(
            name=dev["name"], device_type=dev["type"],
            manufacturer="Cisco", site=dev["site"],
            role="Network Device", status="active")))
        for iface in dev["interfaces"]:
            entities.append(Entity(interface=Interface(
                device=dev["name"], name=iface, type="1000base-t")))

    response = client.ingest(entities=entities, metadata={
        "scan_id": "discovery-2026-01-15", "source": "network_scanner"})
```

## Nested Objects with Full Control

```python
from netboxlabs.diode.sdk.ingester import Device, Site, DeviceType, Manufacturer

device = Device(
    name="router-01",
    device_type=DeviceType(model="ISR 4451", manufacturer=Manufacturer(name="Cisco")),
    site=Site(name="Chicago-DC", status="active",
              metadata={"region": "us-central", "tier": "tier-1"}),
    role="Core Router",
    status="active",
)
```

## Metadata

```python
# Entity-level
device = Device(name="switch-01", device_type="C9300", site="NYC",
    metadata={"discovered_by": "scanner", "confidence_score": 0.95})

# Request-level
response = client.ingest(entities=[Entity(device=device)],
    metadata={"batch_id": "scan-001", "source_system": "scanner"})
```

## Dry Run

Test without contacting the server:

```python
from netboxlabs.diode.sdk import DiodeDryRunClient

with DiodeDryRunClient(app_name="my_app", output_dir="/tmp") as client:
    device = Device(name="test-switch", device_type="Test Type", site="Test Site")
    client.ingest([Entity(device=device)])
    # Creates /tmp/my_app_<timestamp>.json for review
```

Replay: `diode-replay-dryrun --file /tmp/my_app_*.json --target grpc://diode.example.com:8080/diode --app-name my-test-app --app-version 0.0.1`

## Authentication

Diode uses OAuth2 client credentials. Generate in NetBox UI under **Diode > Client Credentials**.

```python
import os
os.environ["DIODE_CLIENT_ID"] = "your-client-id"
os.environ["DIODE_CLIENT_SECRET"] = "your-client-secret"

with DiodeClient(target="grpcs://diode.example.com/diode",
                 app_name="my-app", app_version="1.0.0") as client:
    ...
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DIODE_CLIENT_ID` | OAuth2 client ID |
| `DIODE_CLIENT_SECRET` | OAuth2 client secret |
| `DIODE_CERT_FILE` | Custom CA certificate path |
| `DIODE_SKIP_TLS_VERIFY` | Skip TLS verification (dev only) |
| `DIODE_SDK_LOG_LEVEL` | Log level (default: INFO) |
| `HTTPS_PROXY` / `HTTP_PROXY` | Proxy configuration |
| `NO_PROXY` | Hosts to bypass proxy |

## Supported Entity Types

**DCIM**: Device, DeviceType, DeviceBay, Interface, ConsolePort, PowerPort, Rack, Site, Location, Manufacturer, Platform, Module, Cable
**IPAM**: IPAddress, Prefix, VLAN, VLANGroup, VRF, ASN, Aggregate
**Virtualization**: VirtualMachine, VMInterface, Cluster, ClusterType
**Circuits**: Circuit, CircuitType, Provider, ProviderNetwork
**Tenancy**: Tenant, TenantGroup, Contact, ContactRole
And 70+ more total.

## Architecture

```
Your Script (Diode SDK) → gRPC → Diode Server → REST → NetBox
                                      ↓
                            Resolves dependencies
                            Creates missing objects
                            Handles ordering
```

## References

- [Diode Server](https://github.com/netboxlabs/diode)
- [Diode Python SDK](https://github.com/netboxlabs/diode-sdk-python)
- [Diode Go SDK](https://github.com/netboxlabs/diode-sdk-go)
- [Diode NetBox Plugin](https://github.com/netboxlabs/diode-netbox-plugin)
- [Getting Started](https://github.com/netboxlabs/diode/blob/develop/GET_STARTED.md)
