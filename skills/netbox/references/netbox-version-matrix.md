# NetBox Version Compatibility Matrix

Quick reference for version-dependent features across the platform.

> **Current line:** NetBox 4.6.x (this matrix covers **4.5–4.6**). 4.6 moved the platform to **Django 6.0** (4.5 was Django 5.2), Python **3.12–3.14**, and **deprecated PostgreSQL 14** (PG 15+ required from a future v4.7). Run **≥4.6.1** for the CVE-2026-29514 (template `environment_params` RCE) fix.

## NetBox Core

| Feature | Version | Notes |
|---------|---------|-------|
| REST API | All | Stable, versioned |
| GraphQL API | 2.9+ | Strawberry-based since 4.0 |
| v2 Tokens (`nbt_` prefix) | 4.5+ | Use these! Plaintext shown once at creation on 4.6.1+ |
| v1 Token (`Token <plaintext>`) | Deprecated 4.6 | **Removed in v5.0** (not 4.7) — migrate to v2 |
| Cursor-based GraphQL pagination | 4.5.2+ | `pagination: {start: N, limit: M}` |
| Cursor-based REST pagination | 4.6+ | `?start=<id>` — prefer over deep `?offset=` |
| `add_tags` / `remove_tags` write-only fields | 4.6+ | Avoid read-modify-write clobber of `tags` |
| ETag / `If-Match` optimistic concurrency | 4.6+ | 412 on conflict → re-fetch and retry |
| New models: CableBundle, RackGroup, VirtualMachineType | 4.6+ | See data-modeling skill |
| Custom field `validation_schema` (JSON Schema) | 4.6+ | Validate JSON custom fields |
| Custom Objects | 4.4+ | Via `netbox-custom-objects`; current plugin v0.5.1 needs **4.5.2+** |
| Job framework | 4.0+ | Replaced older task queue |
| Config templates (Jinja2) | 3.5+ | Assigned to devices/roles; sandbox hardened in 4.6.1 |
| Custom validators | 3.4+ | `CUSTOM_VALIDATORS` config |
| Event rules (replaces webhooks) | 4.0+ | Unified event handling; payload `request` object added 4.6 |

## Platform Products

| Product | Plugin / NetBox | Notes |
|---------|---------------|-------|
| NetBox Branching | v1.0.3 · NetBox 4.4.1–4.6 | Requires PostgreSQL schema support; v1.0 added `migrate` branch action |
| NetBox Changes | v1.0.1 · NetBox 4.4–4.6 | Pairs with Branching; v1.0 allows multiple CRs per branch (one active) |
| NetBox Custom Objects | v0.5.1 · NetBox 4.5.2–4.6 | No-code extensibility; v0.5 added polymorphic refs + `on_delete_behavior` |
| NetBox Validation | NetBox 4.2+ | Policy-based compliance checks |
| NetBox Asset Lifecycle | NetBox 4.5–4.6 | Public Preview; procurement + spares |
| NetBox Data Exchange (NDX) | Cloud / Enterprise | In-product feature; type-definition catalog (YAML) is open and version-independent |
| Diode | NetBox 4.2.3+ | gRPC ingestion service |
| Orb Agent (Discovery) | Agent v2.9.0 · NetBox 4.2+ | Via Diode |
| NetBox Assurance | Plugin v1.5.3 · NetBox 4.4.10–4.6 | Licensed add-on for NetBox Cloud and Enterprise; fed by Diode/Discovery |
| netbox-mcp-server | NetBox 4.5+ | v2 tokens required |
| Platform MCP Server | NetBox 4.5+ | Hosted on NetBox Cloud; v2 tokens (`nbt_`) required |
| NetBox Cloud | Always latest | Managed by NetBox Labs |

## SDK Versions

| SDK | Current | Language | NetBox Compatibility |
|-----|---------|----------|---------------------|
| pynetbox | 7.x | Python | All versions |
| diode-sdk-python | 1.12.0 | Python | 4.2.3+ |
| diode-sdk-go | 1.9.0 | Go | 4.2.3+ (import `.../diode-sdk-go/diode`) |
| netbox-graphql-query-optimizer | 1.x | Python | 4.0+ |
