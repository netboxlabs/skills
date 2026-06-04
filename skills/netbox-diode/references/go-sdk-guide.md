# Go SDK Guide

## Installation

```bash
go get github.com/netboxlabs/diode-sdk-go
```

**Requirements:** Go 1.24+ (diode-sdk-go v1.9.0)

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

Same environment variables as Python (`DIODE_CLIENT_ID`, `DIODE_CLIENT_SECRET`, etc.) are also supported.

## Entity Construction

Go uses struct types with pointer fields. Use helper functions for primitive values:

### Pointer Helpers

`diode.String()`, `diode.Int()`, `diode.Int64()`, `diode.Float64()`, `diode.Bool()`

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
// Automatic chunking (0 = default 3MB)
resp, err := client.Ingest(ctx, entities, diode.WithChunking(0))

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
