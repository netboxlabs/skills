# Python SDK Guide

## Installation

```bash
pip install netboxlabs-diode-sdk
```

**Requirements:** Python 3.10+, NetBox 4.5+ (covers 4.5.x–4.6.x); diode-sdk-python v1.12.0

## Client Setup

### Standard Client

```python
from netboxlabs.diode.sdk import DiodeClient

with DiodeClient(
    target="grpc://localhost:8080/diode",  # grpc:// = insecure, grpcs:// = TLS
    app_name="my-app",
    app_version="1.0.0",
    # Credentials via params or env vars DIODE_CLIENT_ID / DIODE_CLIENT_SECRET
) as client:
    pass
```

### Constructor Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `target` | Yes | Diode server address. `grpc://` or `http://` = insecure; `grpcs://` or `https://` = TLS |
| `app_name` | Yes | Producer application name (sent in metadata) |
| `app_version` | Yes | Producer application version |
| `client_id` | Yes* | OAuth2 client ID (*or use `DIODE_CLIENT_ID` env var) |
| `client_secret` | Yes* | OAuth2 client secret (*or use `DIODE_CLIENT_SECRET` env var) |
| `max_auth_retries` | No | Auto-retry count on auth failure (default: 3) |
| `cert_file` | No | Custom TLS certificate path (or `DIODE_CERT_FILE` env var) |

### Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `DIODE_CLIENT_ID` | OAuth2 client ID | (required) |
| `DIODE_CLIENT_SECRET` | OAuth2 client secret | (required) |
| `DIODE_SDK_LOG_LEVEL` | Log level | INFO |
| `DIODE_CERT_FILE` | Custom TLS certificate path | (none) |
| `DIODE_SKIP_TLS_VERIFY` | Skip TLS verification (dev only) | false |
| `DIODE_SENTRY_DSN` | Sentry error reporting | (none) |
| `DIODE_DRY_RUN_OUTPUT_DIR` | Dry run output directory | (none) |
| `DIODE_MAX_AUTH_RETRIES` | Max auth retry attempts | 3 |

## Entity Construction

### Imports

```python
from netboxlabs.diode.sdk.ingester import (
    Device, DeviceType, DeviceRole, Interface, IPAddress,
    Prefix, Site, Manufacturer, Platform, VLAN, VRF,
    Tenant, Cluster, VirtualMachine, Entity,
)
```

### String Shorthand

Pass a string for any nested reference that has a primary value mapping:

```python
device = Device(
    name="sw-01",
    device_type="Catalyst 9300",   # → DeviceType(model="Catalyst 9300")
    manufacturer="Cisco",          # → Manufacturer(name="Cisco")
    site="NYC-DC1",                # → Site(name="NYC-DC1")
    role="Access Switch",          # → DeviceRole(name="Access Switch")
    status="active",
)
```

### Full Object Form (when you need extra attributes)

```python
device = Device(
    name="sw-01",
    device_type=DeviceType(model="Catalyst 9300", part_number="C9300-48T"),
    site=Site(name="NYC-DC1", status="active", region="US-East"),
    role="Access Switch",
)
```

### Metadata

```python
# Entity-level metadata
device = Device(name="sw-01", site="NYC", metadata={"source": "nmap"})

# Request-level metadata
response = client.ingest(entities=entities, metadata={"batch_id": "001"})
```

## Ingestion

### Single and Batch

```python
# Single
response = client.ingest(entities=[Entity(device=device)])

# Batch — multiple entity types in one call
response = client.ingest(entities=[
    Entity(device=device1),
    Entity(device=device2),
    Entity(interface=iface1),
    Entity(ip_address=ip1),
])
```

### Stream Parameter

```python
response = client.ingest(entities=entities, stream="my-stream")
# Default stream is "latest"
```

### Chunked Ingestion

For batches that may exceed gRPC's ~4MB message limit:

```python
from netboxlabs.diode.sdk import create_message_chunks

for chunk in create_message_chunks(entities):  # default 3MB chunks
    client.ingest(entities=chunk)

# Custom chunk size
for chunk in create_message_chunks(entities, max_chunk_size_mb=2.0):
    client.ingest(entities=chunk)
```

## Error Handling

```python
from netboxlabs.diode.sdk.exceptions import DiodeClientError, DiodeConfigError

try:
    with DiodeClient(target=target, app_name="app", app_version="1.0") as client:
        response = client.ingest(entities=entities)
        if response.errors:
            # Per-entity validation errors (not transport errors)
            for err in response.errors:
                log.warning(f"Entity rejected: {err}")
except DiodeConfigError as e:
    # Auth/config: bad credentials, unreachable endpoint
    log.error(f"Config error: {e}")
except DiodeClientError as e:
    # gRPC transport error
    log.error(f"gRPC {e.status_code}: {e.details}")
```

## Dry Run Client

Test entity construction without a running server:

```python
from netboxlabs.diode.sdk import DiodeDryRunClient

with DiodeDryRunClient(app_name="my_app", output_dir="/tmp") as client:
    client.ingest([Entity(device=device)])
    # Writes /tmp/my_app_<timestamp_ns>.json
```

### Replay Dry Run Files

```python
from netboxlabs.diode.sdk import DiodeClient, load_dryrun_entities

entities = list(load_dryrun_entities("my_app_12345.json"))
with DiodeClient(target=target, app_name="app", app_version="1.0") as client:
    client.ingest(entities=entities)
```

Or via CLI:
```bash
diode-replay-dryrun --file /tmp/file.json --target grpc://... --app-name app --app-version 1.0
```

## OTLP Client

Export entities as OpenTelemetry log records:

```python
from netboxlabs.diode.sdk import DiodeOTLPClient

with DiodeOTLPClient(
    target="grpc://localhost:4317",
    app_name="my-producer",
    app_version="0.0.1",
) as client:
    client.ingest([Entity(site="Site1")])
```

## Complete Discovery Example

```python
from netboxlabs.diode.sdk import DiodeClient, create_message_chunks
from netboxlabs.diode.sdk.ingester import Device, Interface, IPAddress, Entity
from netboxlabs.diode.sdk.exceptions import DiodeClientError, DiodeConfigError

def push_discovered_devices(discovered_hosts: list[dict]):
    entities = []
    for host in discovered_hosts:
        entities.append(Entity(device=Device(
            name=host["hostname"],
            device_type=host.get("model", "Unknown"),
            manufacturer=host.get("vendor", "Unknown"),
            site=host["site"],
            role="Network Device",
            status="active",
            serial=host.get("serial"),
        )))
        for iface in host.get("interfaces", []):
            entities.append(Entity(interface=Interface(
                device=host["hostname"],
                name=iface["name"],
                type=iface.get("type", "other"),
            )))
            if iface.get("ip"):
                entities.append(Entity(ip_address=IPAddress(
                    address=iface["ip"],  # Must be CIDR: "10.0.1.1/24"
                )))

    try:
        with DiodeClient(
            target="grpcs://diode.example.com/diode",
            app_name="network-discovery",
            app_version="2.0.0",
        ) as client:
            for chunk in create_message_chunks(entities):
                response = client.ingest(entities=chunk, metadata={
                    "source": "network-scanner",
                })
                if response.errors:
                    for err in response.errors:
                        print(f"Warning: {err}")
    except DiodeConfigError as e:
        print(f"Auth failed: {e}")
    except DiodeClientError as e:
        print(f"Ingestion failed: {e.status_code} - {e.details}")
```
