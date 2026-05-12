# NetBox Labs Product Catalog

Reference for all platform products and their interfaces.

## Products

| Product | Category | Primary Interface | Skill |
|---------|----------|-------------------|-------|
| **NetBox** (OSS) | Source of Truth | REST API, GraphQL | `netbox-api-integration`, `netbox-data-modeling` |
| **NetBox Branching** | Change Management | REST API (X-NetBox-Branch header) | `netbox-branching` |
| **NetBox Changes** | Change Management | REST API | `netbox-changes` |
| **Diode** | Data Ingestion | gRPC (Python/Go SDKs) | `netbox-diode` |
| **Orb Agent** | Discovery | Config file (YAML) | `netbox-discovery` |
| **NetBox Assurance** | Drift Detection | REST API (via Diode reconciler) | `netbox-assurance` |
| **NetBox Designs** | Design Automation | REST API, CLI (`nbd`) | *(no skill yet)* |
| **Platform MCP Server** | Agent Interface | MCP protocol (SSE/stdio) | `netbox-mcp-server` |
| **Agent Platform** | Agent Runtime | REST API | *(no skill yet)* |
| **NetBox Cloud** | Managed Deployment | Console UI, REST API | *(no skill yet)* |
| **Copilot** | Interactive AI | Chat interface | (future skill) |
| **Visual Explorer** | Visualization | Micro-frontend | (no skill needed — UI only) |

## Product Relationships

```
                    ┌──────────────┐
                    │   NetBox     │ ← Source of Truth
                    │  (REST/GQL) │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴──────┐
    │ Branching  │   │   Diode   │   │  Designs   │
    │ + Changes  │   │ (ingest)  │   │ (automate) │
    └────────────┘   └─────┬─────┘   └────────────┘
                           │
                    ┌──────┴───────┐
                    │  Orb Agent   │
                    │ (discovery)  │
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │  Assurance   │
                    │ (drift)      │
                    └──────────────┘

    ┌──────────────┐
    │ MCP Server   │ ← Agent ↔ NetBox interface
    └──────┬───────┘
           │
    ┌──────┴───────┐
    │Agent Platform│ ← Agent runtime
    │ + Copilot    │
    └──────────────┘
```
