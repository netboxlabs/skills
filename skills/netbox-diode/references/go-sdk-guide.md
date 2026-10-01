# Go SDK Guide

## Installation

```bash
go get github.com/netboxlabs/diode-sdk-go
```

**Requirements:** Go 1.25+ (diode-sdk-go v1.12.0; entities regenerated for NetBox 4.7.0 in 1.12.0)

## Client Setup

```go
import diode "github.com/netboxlabs/diode-sdk-go/diode"  // exported types live in the diode/ subpackage

client, err := diode.NewClient(
    "grpc://localhost:8080/diode",  // grpc:// = insecure, grpcs:// = TLS
    "my-app",
    "1.0.0",
    diode.WithClientID("YOUR_CLIENT_ID"),
    diode.WithClientSecret("YOUR_CLIENT_SECRET"),
)
if err != nil {
    log.Fatal(err)
}
defer client.Close()
```

### Functional Options

| Option | Description |
|--------|-------------|
| `WithClientID(id)` | OAuth2 client ID |
| `WithClientSecret(secret)` | OAuth2 client secret |
| `WithCertFile(path)` | Custom TLS certificate |
| `WithSkipTLSVerify()` | Skip TLS verification (dev only) |

Same environment variables as Python (`DIODE_CLIENT_ID`, `DIODE_CLIENT_SECRET`, `DIODE_MAX_AUTH_RETRIES`, `DIODE_CERT_FILE`, `DIODE_SKIP_TLS_VERIFY`, etc.) are also supported. `NewClient` authenticates immediately and returns an error if the token request fails.

### Authentication Behaviour

| Version | Behaviour |
|---------|-----------|
| 1.10+ | Token request retries on `429`/`500`/`502`/`503` with exponential backoff (1s → 30s cap, jitter), honouring `Retry-After` on `429`/`503`; other statuses fail immediately. Backoff respects the `ctx` passed to `Ingest`. |
| 1.11.1+ | The access token is refreshed about one minute before its `expires_in` lifetime ends, so a long-running producer never sends an expired token. Concurrent `Ingest` calls share one refresh. An `Ingest` sent with a stale token is retried after refresh regardless of the gRPC status code. |
| all | `DIODE_MAX_AUTH_RETRIES` (default 3) bounds both the token request and re-auth on `Unauthenticated`; values ≤ 0 are rejected at `NewClient`. |

## Entity Construction

Go uses struct types with pointer fields. Use helper functions for primitive values:

### Pointer Helpers

`diode.String()`, `diode.Bool()`, `diode.Int()`, `diode.Int32()`, `diode.Int64()`, `diode.Uint()`, `diode.Uint32()`, `diode.Uint64()`, `diode.Float32()`, `diode.Float64()`. Date fields such as `DeviceType.EndOfLife` are `*time.Time` (no helper — take the address of a value).

### Device with Full References

```go
device := &diode.Device{
    Name:       diode.String("sw-01"),
    DeviceType: &diode.DeviceType{
        Model:        diode.String("Catalyst 9300"),
        Manufacturer: &diode.Manufacturer{Name: diode.String("Cisco")},
    },
    Site:   &diode.Site{Name: diode.String("NYC-DC1")},
    Role:   &diode.DeviceRole{Name: diode.String("Access Switch")},
    Status: diode.String("active"),
    Serial: diode.String("ABC123"),
}
```

> **No string shorthand in Go.** Every nested reference must be a full struct. You cannot pass `"Cisco"` for a manufacturer field — use `&diode.Manufacturer{Name: diode.String("Cisco")}`.

### Interface

```go
iface := &diode.Interface{
    Device: &diode.Device{Name: diode.String("sw-01")},
    Name:   diode.String("Gi0/1"),
    Type:   diode.String("1000base-t"),
}
```

### IPAddress

```go
ip := &diode.IPAddress{
    Address: diode.String("10.0.1.1/24"),  // Must be CIDR
}
```

### NetBox 4.7 Fields (1.12.0+)

Only land against NetBox 4.7 with a matching Diode plugin:

```go
eol := time.Date(2030, 12, 31, 0, 0, 0, 0, time.UTC)
entities := []diode.Entity{
    &diode.DeviceType{
        Model:         diode.String("Catalyst 9300"),
        Manufacturer:  &diode.Manufacturer{Name: diode.String("Cisco")},
        CoolingMethod: diode.String("air"),
        EndOfLife:     &eol,
    },
    &diode.Interface{ // channelized parent + one channel subinterface
        Device: &diode.Device{Name: diode.String("sw-01")}, Name: diode.String("Ethernet1"),
        Type: diode.String("100gbase-x-qsfp28"), Channels: diode.Int64(4),
    },
    &diode.Interface{
        Device: &diode.Device{Name: diode.String("sw-01")}, Name: diode.String("Ethernet1/1"),
        Type: diode.String("channel"), ChannelId: diode.Int64(1),
        Parent: &diode.Interface{Device: &diode.Device{Name: diode.String("sw-01")}, Name: diode.String("Ethernet1")},
        MacAddress: diode.String("00:11:22:33:44:55"), // sets the primary MAC directly
    },
    &diode.Service{ // multi-protocol; do not set Protocol/Ports alongside
        Device: &diode.Device{Name: diode.String("dns-01")}, Name: diode.String("dns"),
        PortMappings: []string{"tcp/53", "udp/53"},
    },
    &diode.CoolingFeed{
        Name: diode.String("CDU1-Loop-A"), Status: diode.String("active"),
        CoolingSource: &diode.CoolingSource{Name: diode.String("CDU-1")},
        Rack:          &diode.Rack{Name: diode.String("R101")},
    },
}
```

`ModuleType.ModuleBayTypes` / `ModuleBay.ModuleBayTypes` take `[]*diode.ModuleBayType`; `RackReservation.User` takes `*diode.User{Username: ...}`. Full list in [entity-catalog.md](entity-catalog.md#netbox-47-additions).

## Ingestion

### Basic Ingestion

```go
entities := []diode.Entity{device, iface, ip}
resp, err := client.Ingest(context.Background(), entities)
if err != nil {
    log.Fatal(err)
}
if resp != nil && resp.Errors != nil {
    log.Printf("Entity errors: %v", resp.Errors)
}
```

### With Metadata

```go
resp, err := client.Ingest(ctx, entities,
    diode.WithIngestMetadata(diode.Metadata{
        "batch_id": "scan-001",
        "source":   "network-scanner",
    }),
)
```

### Chunked Ingestion

```go
// Automatic chunking (0 = default 3MB). Only the LAST chunk's response is returned
// unless WithChunkingReturnAllResults() is added.
resp, err := client.Ingest(ctx, entities, diode.WithChunking(0), diode.WithChunkingReturnAllResults())

// Manual chunking
chunks := diode.CreateMessageChunks(protoEntities, 3.5) // 3.5 MB
for _, chunk := range chunks {
    resp, err := client.IngestProto(ctx, chunk)
    if err != nil {
        log.Printf("Chunk error: %v", err)
    }
}
```

## Dry Run Client

```go
client, err := diode.NewDryRunClient("my-app", "/tmp/output")
if err != nil {
    log.Fatal(err)
}
defer client.Close()

resp, err := client.Ingest(ctx, entities)
// Writes JSON to /tmp/output/
```

## OTLP Client

```go
client, err := diode.NewOTLPClient(
    "grpc://localhost:4317",
    "my-producer",
    "0.0.1",
)
```

## Complete Example

```go
package main

import (
    "context"
    "log"
    diode "github.com/netboxlabs/diode-sdk-go/diode"
)

func main() {
    client, err := diode.NewClient(
        "grpcs://diode.example.com/diode",
        "network-discovery",
        "1.0.0",
        diode.WithClientID("my-client-id"),
        diode.WithClientSecret("my-client-secret"),
    )
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }
    defer client.Close()

    entities := []diode.Entity{
        &diode.Device{
            Name:       diode.String("router-01"),
            DeviceType: &diode.DeviceType{Model: diode.String("ISR 4451")},
            Site:       &diode.Site{Name: diode.String("Chicago-DC")},
            Role:       &diode.DeviceRole{Name: diode.String("Core Router")},
            Status:     diode.String("active"),
        },
        &diode.Interface{
            Device: &diode.Device{Name: diode.String("router-01")},
            Name:   diode.String("GigabitEthernet0/0"),
            Type:   diode.String("1000base-t"),
        },
        &diode.IPAddress{
            Address: diode.String("10.0.0.1/30"),
        },
    }

    resp, err := client.Ingest(context.Background(), entities,
        diode.WithChunking(0),
    )
    if err != nil {
        log.Fatalf("Ingestion failed: %v", err)
    }
    if resp != nil && resp.Errors != nil {
        log.Printf("Entity errors: %v", resp.Errors)
    }
    log.Println("Ingestion complete")
}
```
