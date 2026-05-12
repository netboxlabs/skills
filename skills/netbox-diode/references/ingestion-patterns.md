# Ingestion Patterns

Advanced patterns for Diode data ingestion: chunking, dry run, OTLP export, and metadata strategies.

## Batch Ingestion

Send multiple entities of mixed types in a single call:

```python
entities = [
    Entity(device=device1),
    Entity(device=device2),
    Entity(interface=iface1),
    Entity(ip_address=ip1),
]
response = client.ingest(entities=entities)
```

Entity order is preserved but does not affect reconciliation — the reconciler handles dependency resolution regardless of order.

## Chunked Ingestion

gRPC enforces a ~4MB message size limit. For large batches, use built-in chunking.

### Python

```python
from netboxlabs.diode.sdk import create_message_chunks

# Default: 3MB chunks (safe margin under 4MB limit)
for chunk in create_message_chunks(entities):
    client.ingest(entities=chunk)

# Custom chunk size
for chunk in create_message_chunks(entities, max_chunk_size_mb=2.0):
    client.ingest(entities=chunk)
```

### Go

```go
// Automatic chunking via option
resp, err := client.Ingest(ctx, entities, diode.WithChunking(0))  // 0 = default 3MB

// Manual chunking for more control
chunks := diode.CreateMessageChunks(protoEntities, 3.5)
for _, chunk := range chunks {
    resp, err := client.IngestProto(ctx, chunk)
}
```

Chunking uses greedy bin-packing: entities accumulate until the next would exceed the limit, then a new chunk starts. Entity order is preserved within and across chunks.

## Metadata

### Entity-Level Metadata

Attach context to individual entities:

```python
device = Device(
    name="sw-01",
    site="NYC",
    metadata={"discovered_by": "nmap", "confidence": 0.95},
)
```

### Request-Level Metadata

Attach context to an entire ingestion batch:

```python
response = client.ingest(
    entities=entities,
    metadata={"batch_id": "import-001", "source": "scanner"},
)
```

Go uses `diode.WithIngestMetadata(diode.Metadata{...})`.

> **Metadata values must be JSON-compatible types:** strings, numbers, booleans, lists, dicts. No datetime objects or custom classes.

## Stream Parameter

The `stream` parameter identifies the data stream for the reconciler (default: `"latest"`):

```python
response = client.ingest(entities=entities, stream="campus-network")
```

## Dry Run Mode

Test entity construction and serialization without a running Diode server.

### Python

```python
from netboxlabs.diode.sdk import DiodeDryRunClient

with DiodeDryRunClient(app_name="my_app", output_dir="/tmp") as client:
    client.ingest([Entity(device=device)])
    # Creates /tmp/my_app_<timestamp_ns>.json
```

Or set `DIODE_DRY_RUN_OUTPUT_DIR` env var to enable dry-run output globally.

### Replay Dry Run Files

**Python API:**
```python
from netboxlabs.diode.sdk import load_dryrun_entities

entities = list(load_dryrun_entities("my_app_12345.json"))
client.ingest(entities=entities)
```

**CLI:**
```bash
diode-replay-dryrun \
    --file /tmp/my_app_12345.json \
    --target grpc://diode.example.com:8080/diode \
    --app-name my-app \
    --app-version 1.0
```

### Go Dry Run

```go
client, err := diode.NewDryRunClient("my-app", "/tmp/output")
```

## OTLP Export

Export entities as OpenTelemetry log records for collector-based pipelines:

### Python

```python
from netboxlabs.diode.sdk import DiodeOTLPClient

with DiodeOTLPClient(
    target="grpc://localhost:4317",
    app_name="my-producer",
    app_version="0.0.1",
) as client:
    client.ingest([Entity(site="Site1")])
```

Each entity becomes a JSON log record with producer metadata (`app_name`, `app_version`). Useful when you have an existing OpenTelemetry Collector pipeline and want to route Diode data through it.

### Go OTLP

```go
client, err := diode.NewOTLPClient("grpc://localhost:4317", "my-producer", "0.0.1")
```
