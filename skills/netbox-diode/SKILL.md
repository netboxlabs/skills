---
name: netbox-diode
description: >
  Patterns for ingesting data into NetBox using the Diode SDK (Python and Go).
  Use when building network discovery, bulk import, or automated data population
  integrations that write data to NetBox via gRPC. Covers SDK setup, entity
  construction, batch ingestion, chunking, reconciler behavior, error handling,
  and when to use Diode vs the REST API.
license: Apache-2.0
---

# NetBox Diode SDK

Diode is a gRPC-based data ingestion service for NetBox. Instead of managing dependency order and object IDs via the REST API, you describe objects by name and the Diode reconciler resolves dependencies, creates missing objects, and performs upserts automatically.

**This skill covers the open-source Python and Go SDKs** (diode-sdk-python v1.12.0 / diode-sdk-go v1.9.0), targeting NetBox **4.5.x–4.6.x**. The Diode server and reconciler are proprietary — this skill describes their observable behavior, not internals.

> **Your knowledge of Diode SDK may be outdated.** Entity types, SDK methods, and reconciler behavior evolve between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Diode docs | `https://netboxlabs.com/docs/diode/` | Architecture, getting started |
| Diode Python SDK | `https://netboxlabs.com/docs/diode/sdk/python/` | Python SDK reference |
| Python SDK repo | `https://github.com/netboxlabs/diode-sdk-python` | Source, entity types, changelog |
| Go SDK repo | `https://github.com/netboxlabs/diode-sdk-go` | Source, entity types, changelog |
| Protobuf reference | `https://netboxlabs.com/docs/diode/diode-proto/` | Entity schema definitions |
| NetBox MCP server | If configured — verify objects created by Diode landed correctly | Post-ingestion validation |

For a quick comparison of Diode vs REST API, see
[netbox-api-integration/references/diode-integration.md](../netbox-api-integration/references/diode-integration.md).

## FIRST: Verify Setup

Confirm the Diode SDK is installed and your credentials are configured:

```bash
# Python
pip show netboxlabs-diode-sdk

# Environment variables (recommended)
echo $DIODE_CLIENT_ID   # Should be set
echo $DIODE_CLIENT_SECRET  # Should be set
```

If the SDK is not installed: `pip install netboxlabs-diode-sdk`

To verify Diode server connectivity:
```python
from netboxlabs.diode.sdk import DiodeClient
with DiodeClient(target="grpc://YOUR_DIODE_HOST:8080/diode", app_name="test", app_version="0.1") as client:
    print("Connected")  # Will raise DiodeConfigError if auth fails
```

---

## When to Use Diode

| Scenario | Recommended |
|----------|-------------|
| Network discovery pushing many objects | **Diode** |
| Bulk import / migration | **Diode** |
| Creating many related objects (devices + interfaces + IPs) | **Diode** |
| Reading / querying NetBox data | REST or GraphQL (Diode is write-only) |
| Single object CRUD with immediate read-back | REST API |
| Complex filtered searches | REST or GraphQL |

**Key advantages over REST:** No dependency ordering, no ID lookups, automatic object creation, gRPC performance for high volume.

**Key limitation:** Write-only. You cannot query NetBox through Diode.

## Quick Reference

### Client Types

| Client | Python | Go | Purpose |
|--------|--------|----|---------|
| Standard | `DiodeClient` | `diode.NewClient` | Production ingestion (requires Diode server) |
| Dry Run | `DiodeDryRunClient` | `diode.NewDryRunClient` | JSON output for testing (no server needed) |
| OTLP | `DiodeOTLPClient` | `diode.NewOTLPClient` | Export as OpenTelemetry log records |

### Authentication

OAuth2 client credentials via environment variables (recommended):

```bash
export DIODE_CLIENT_ID="your-client-id"
export DIODE_CLIENT_SECRET="your-client-secret"
```

Or pass directly: `client_id=` / `client_secret=` (Python), `WithClientID()` / `WithClientSecret()` (Go).

### Minimal Python Example

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Interface, IPAddress, Entity

with DiodeClient(
    target="grpc://localhost:8080/diode",
    app_name="my-discovery",
    app_version="1.0.0",
) as client:
    entities = [
        Entity(device=Device(
            name="sw-01",
            device_type="Catalyst 9300",
            manufacturer="Cisco",
            site="NYC-DC1",
            role="Access Switch",
            status="active",
        )),
        Entity(interface=Interface(device="sw-01", name="Gi0/1", type="1000base-t")),
        Entity(ip_address=IPAddress(address="10.0.1.1/24")),
    ]
    response = client.ingest(entities=entities)
    if response.errors:
        for err in response.errors:
            print(f"Error: {err}")
```

### Minimal Go Example

```go
import diode "github.com/netboxlabs/diode-sdk-go/diode"  // exported types live in the diode/ subpackage

client, err := diode.NewClient(
    "grpc://localhost:8080/diode",
    "my-discovery",
    "1.0.0",
    diode.WithClientID("YOUR_CLIENT_ID"),
    diode.WithClientSecret("YOUR_CLIENT_SECRET"),
)
if err != nil { log.Fatal(err) }
defer client.Close()

entities := []diode.Entity{
    &diode.Device{
        Name:       diode.String("sw-01"),
        DeviceType: &diode.DeviceType{Model: diode.String("Catalyst 9300")},
        Site:       &diode.Site{Name: diode.String("NYC-DC1")},
        Role:       &diode.DeviceRole{Name: diode.String("Access Switch")},
        Status:     diode.String("active"),
    },
}
resp, err := client.Ingest(context.Background(), entities)
```

## Entity Construction

### String Shorthand (Python only)

Python entities with a primary key in `PRIMARY_VALUE_MAP` accept string shorthand for nested references:

```python
# These are equivalent:
Device(name="sw-01", site="NYC-DC1")
Device(name="sw-01", site=Site(name="NYC-DC1"))
```

Use full objects when you need to set attributes beyond the primary key:

```python
Device(name="sw-01", site=Site(name="NYC-DC1", status="active", region="US-East"))
```

> **Go has no string shorthand.** Always use full struct types with pointer helpers (`diode.String()`, `diode.Int()`).

### Required Fields

Most entities only require their primary identifying field (e.g., `name` for Device, `address` for IPAddress). The reconciler creates referenced objects automatically. See [references/entity-catalog.md](references/entity-catalog.md) for the full list.

### Key Uniqueness Constraints

| Entity | Unique By |
|--------|-----------|
| Device | `name` + `site` + `tenant` |
| Interface | `name` + `device` |
| IPAddress | `address` + `vrf` |
| Prefix | `prefix` + `vrf` |
| VLAN | `vid` + `group` |

## Reconciler Behavior

The reconciler is the server-side component that processes ingested data. Key observable behaviors:

1. **Automatic dependency resolution** — referencing `site="NYC-DC1"` creates the Site if it doesn't exist
2. **Name-based matching** — objects matched by their primary identifying field
3. **No ordering required** — unlike REST, you don't need to create manufacturers before device types
4. **Upsert semantics** — matching entities are updated, not duplicated
5. **Slug auto-generation** — `slug` derived from `name` if not provided
6. **Case-sensitive matching** — `"Site ABC"` ≠ `"site abc"`
7. **Producer provenance** — entities tagged with `app_name` / `app_version` for audit

## Ingestion Patterns

### Batch Ingestion

Send multiple entities in one call:

```python
response = client.ingest(entities=[
    Entity(device=device1),
    Entity(device=device2),
    Entity(interface=iface1),
], metadata={"batch_id": "scan-001"})
```

### Chunked Ingestion (large batches)

gRPC has a ~4MB message limit. Use built-in chunking for large payloads:

```python
from netboxlabs.diode.sdk import create_message_chunks

for chunk in create_message_chunks(entities):  # default 3MB chunks
    client.ingest(entities=chunk)
```

Go: `client.Ingest(ctx, entities, diode.WithChunking(0))` (0 = default 3MB).

### Dry Run and OTLP

See [references/ingestion-patterns.md](references/ingestion-patterns.md) for dry run testing, replay, and OTLP export.

## Error Handling

### Python Exception Hierarchy

```
DiodeConfigError          # Bad credentials, unreachable auth endpoint
DiodeClientError          # gRPC transport errors (inherits grpc.RpcError; has .code(), .details())
OTLPClientError           # OTLP export failures
```

> **Note:** `DiodeClientError` inherits from `grpc.RpcError`, so `except grpc.RpcError` also catches it.

### Auto-Retry on Auth Expiry

The SDK automatically retries on `UNAUTHENTICATED` gRPC errors (token expiry), up to 3 times by default (`max_auth_retries` or `DIODE_MAX_AUTH_RETRIES`).

### Recommended Pattern

```python
from netboxlabs.diode.sdk.exceptions import DiodeClientError, DiodeConfigError

try:
    with DiodeClient(target=target, app_name="app", app_version="1.0") as client:
        response = client.ingest(entities=entities)
        if response.errors:  # Per-entity validation errors
            for err in response.errors:
                log.warning(f"Entity error: {err}")
except DiodeConfigError as e:
    log.error(f"Config/auth error: {e}")
except DiodeClientError as e:
    log.error(f"gRPC error: {e.status_code} - {e.details}")
```

## Anti-Patterns

1. **IPAddress requires CIDR** — `"192.168.1.1/24"`, not `"192.168.1.1"`
2. **Interface requires device** — name alone isn't unique without device context
3. **Go has no string shorthand** — every nested reference needs a full struct
4. **Diode is write-only** — use REST/GraphQL to read data back
5. **Case-sensitive matching** — inconsistent casing creates duplicate objects
6. **gRPC 4MB limit** — use chunking for large batches
7. **String shorthand limitations** — can't set nested object attributes (use full objects)
8. **Metadata must be JSON-compatible** — strings, numbers, booleans, lists, dicts only

## References

| File | When to Load |
|------|-------------|
| [references/entity-catalog.md](references/entity-catalog.md) | Need the full list of 104 entity types and their fields |
| [references/python-sdk-guide.md](references/python-sdk-guide.md) | Building a Python integration — setup, patterns, examples |
| [references/go-sdk-guide.md](references/go-sdk-guide.md) | Building a Go integration — setup, patterns, examples |
| [references/ingestion-patterns.md](references/ingestion-patterns.md) | Advanced patterns: chunking, dry run, OTLP, metadata |

### Cross-Skill References

- [netbox-api-integration/references/diode-integration.md](../netbox-api-integration/references/diode-integration.md) — Overview and Diode vs REST comparison
- [netbox-discovery](../netbox-discovery/SKILL.md) — Orb Agent network scanning that feeds data via Diode
- [netbox-assurance](../netbox-assurance/SKILL.md) — Assurance engine that compares Diode-ingested data against intended state
