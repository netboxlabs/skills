# NetBox Version Compatibility Matrix

Quick reference for version-dependent features across the platform.

## NetBox Core

| Feature | Version | Notes |
|---------|---------|-------|
| REST API | All | Stable, versioned |
| GraphQL API | 2.9+ | Strawberry-based since 4.0 |
| v2 Tokens (`nbt_` prefix) | 4.5+ | Use these! |
| v1 Token deprecation | 4.7+ | Migrate before this |
| Cursor-based GraphQL pagination | 4.6+ | Before: offset only |
| Custom Objects | 4.4+ | Via `netbox-custom-objects` plugin |
| Job framework | 4.0+ | Replaced older task queue |
| Config templates (Jinja2) | 3.5+ | Assigned to devices/roles |
| Custom validators | 3.4+ | `CUSTOM_VALIDATORS` config |
| Event rules (replaces webhooks) | 4.0+ | Unified event handling |

## Platform Products

| Product | Minimum NetBox | Notes |
|---------|---------------|-------|
| NetBox Branching | 4.2+ | Requires PostgreSQL schema support |
| NetBox Changes | 4.2+ | Pairs with Branching |
| NetBox Custom Objects | 4.4+ | No-code data model extensibility |
| NetBox Validation | 4.2+ | Policy-based compliance checks |
| Diode | 4.2.3+ | gRPC ingestion service |
| Orb Agent (Discovery) | 4.2+ | Via Diode |
| NetBox Assurance | 4.2+ | Via Diode reconciler |
| netbox-mcp-server | 4.5+ | v2 tokens required |
| NetBox Cloud | Always latest | Managed by NetBox Labs |

## SDK Versions

| SDK | Current | Language | NetBox Compatibility |
|-----|---------|----------|---------------------|
| pynetbox | 7.x | Python | All versions |
| diode-sdk-python | 0.x | Python | 4.2.3+ |
| diode-sdk-go | 0.x | Go | 4.2.3+ |
| netbox-graphql-query-optimizer | 1.x | Python | 4.0+ |
