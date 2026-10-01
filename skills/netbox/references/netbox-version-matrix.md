# NetBox Version Compatibility Matrix

Quick reference for version-dependent features across the platform.

> **Current line:** NetBox 4.7.x (this matrix covers **4.5–4.7**). 4.7 moved the platform to **Django 6.1** (4.6 was Django 6.0, 4.5 was Django 5.2), Python **3.12–3.14**, **PostgreSQL 15+ (required — the upgrade aborts on 14)** with the `ltree` extension, and **Redis 6.0+**. Run **≥4.7.2**: 4.7.0/4.7.1 recorded plaintext API tokens created via background bulk requests in job results, and 4.7.0 database dumps restore without the hierarchy cascade triggers. On the 4.6 line run **≥4.6.1** for the CVE-2026-29514 (template `environment_params` RCE) fix.

## NetBox Core

| Feature | Version | Notes |
|---------|---------|-------|
| REST API | All | Stable, versioned |
| GraphQL API | 2.9+ | Strawberry-based since 4.0 |
| v2 Tokens (`nbt_` prefix) | 4.5+ | Use these! Plaintext shown once at creation on 4.6.1+; client can no longer choose the plaintext on 4.7+ (`token` is read-only) |
| v1 Token (`Token <plaintext>`) | Deprecated 4.6 | **Removed in v5.0** — migrate to v2 |
| Cursor-based GraphQL pagination | 4.5.2+ | `pagination: {start: N, limit: M}` |
| Cursor-based REST pagination | 4.6+ | `?start=<id>` — prefer over deep `?offset=`; pynetbox 7.8+ supports it natively |
| `add_tags` / `remove_tags` write-only fields | 4.6+ | Avoid read-modify-write clobber of `tags` |
| ETag / `If-Match` optimistic concurrency | 4.6+ | 412 on conflict → re-fetch and retry |
| Config context pre-rendered & cached | 4.7+ | Always included in device/VM responses; `?exclude=config_context` is **silently ignored** on 4.7 (still required for performance on 4.5–4.6) |
| Selection custom field values as `{value, label}` | 4.7+ | REST and GraphQL responses; raw value still accepted on write |
| Background bulk REST writes (`?background=true`) | 4.7+ | Returns 202 + job; validation deferred to the worker — poll the job. Never use it to create tokens |
| Per-object bulk error responses | 4.7+ | `{"detail", "errors": [...]}` keyed by `index` on create and by `id` on update — still all-or-none |
| Snapshot-aware event rule conditions | 4.7+ | `changed`/`unchanged`/`regex` operators; `snapshots.prechange.<attr>` paths |
| Webhook context `username` / `request_id` | Removed 4.7 | Use `request.user` / `request.id` (available since 4.6) |
| Hierarchies on PostgreSQL `ltree` (was django-mptt) | 4.7+ | `level` no longer filterable/orderable; `NestedGroupModel` deprecated → `NestedLtreeGroupModel` |
| New models: CoolingSource/Feed/Intake/Outflow, ModuleBayType | 4.7+ | See data-modeling skill |
| Channelized subinterfaces (`channels`, `channel_id`) | 4.7+ | Breakout interfaces modeled natively |
| Service `port_mappings` (replaces `protocol` + `ports`) | 4.7+ | Legacy fields deprecated in REST/GraphQL, removed v5.0 |
| DeviceType / ModuleType `end_of_life` | 4.7+ | Core lifecycle date; NDX enrichment carries the sourced detail |
| Rack `form_factor` / `width` / `outer_*` | Deprecated 4.7 | Removed v5.0 — inferred from a mandatory rack type |
| Core custom scripts | Deprecated 4.7 | Supported through 4.8, removed v5.0 in favor of a dedicated plugin; REST execution needs a write-enabled token |
| `JINJA_FILTERS` (was `JINJA2_FILTERS`) | 4.7+ | Old name works until v5.0; plugins can register filters |
| `housekeeping` management command | Removed 4.7 | Deprecated 4.6; tasks run as system jobs since 4.4 |
| New models: CableBundle, RackGroup, VirtualMachineType | 4.6+ | See data-modeling skill |
| Custom field `validation_schema` (JSON Schema) | 4.6+ | Validate JSON custom fields |
| Custom Objects | 4.4+ | Via `netbox-custom-objects`; current plugin v0.7.0 needs **4.5.2+** (0.6.1+ on 4.7) |
| Job framework | 4.0+ | Replaced older task queue |
| Config templates (Jinja2) | 3.5+ | Assigned to devices/roles; sandbox hardened in 4.6.1 |
| Custom validators | 3.4+ | `CUSTOM_VALIDATORS` config |
| Event rules (replaces webhooks) | 4.0+ | Unified event handling; payload `request` object added 4.6; plugin-defined action types 4.7 |

## Platform Products

| Product | Plugin / NetBox | Notes |
|---------|---------------|-------|
| NetBox Branching | v1.2.1 · NetBox **4.7** only; v1.1.3 · NetBox 4.4.1–4.6 | Two supported lines — pick by NetBox version. v1.2 replicates ltree/denormalization triggers into branch schemas; v1.1.2 added `auto_archive_days` |
| NetBox Changes | v1.1.3 · NetBox 4.5.2–4.7 | Pairs with Branching (1.1.x on ≤4.6, 1.2.x on 4.7); v1.1 adds a default policy (pre-selected in the UI) and per-policy independent review |
| NetBox Custom Objects | v0.7.0 · NetBox 4.5.2–4.7 (0.6.1+ for 4.7) | No-code extensibility; v0.6 added branching + read-only GraphQL, v0.7 the related "Custom Objects" tab and YAML schema export |
| NetBox Validation | v1.14.1 · NetBox 4.4–4.7 (4.7 from 1.14.0) | Policy-based compliance checks; run lifecycle hardened in 1.12 |
| NetBox Asset Lifecycle | v0.3.1 · NetBox 4.5.4–4.7 | v0.3 renamed `bom-objects/` → `assets/` and reinitialized migrations (no upgrade from 0.2.x) |
| NetBox Data Exchange (NDX) | Cloud / Enterprise | In-product feature; type-definition catalog (YAML) is open and version-independent |
| Diode | NetBox 4.2.3+ | gRPC ingestion service |
| Orb Agent (Discovery) | Agent v2.15.0 · NetBox 4.2+ | Via Diode; v2.15 ships SNMP and gNMI telemetry backends alongside discovery |
| NetBox Assurance | Plugin v1.5.x · NetBox 4.4.10+ | Licensed add-on for NetBox Cloud and Enterprise; fed by Diode/Discovery — confirm the current ceiling in the product docs |
| netbox-mcp-server | v1.2.1 · NetBox 4.5+ | v2 tokens required; optional bearer auth on the HTTP transport |
| Platform MCP Server | NetBox 4.5+ | Hosted on NetBox Cloud; v2 tokens (`nbt_`) required |
| NetBox Cloud | Always latest | Managed by NetBox Labs |

## SDK Versions

| SDK | Current | Language | NetBox Compatibility |
|-----|---------|----------|---------------------|
| pynetbox | 7.8.0 | Python | All versions; 7.7 adds 4.6 models, 7.8 adds cursor pagination + branching/custom-objects integrations |
| diode-sdk-python | 1.14.1 | Python | 4.2.3+; ingester regenerated for 4.7 in 1.14.0 |
| diode-sdk-go | 1.12.0 | Go | 4.2.3+ (import `.../diode-sdk-go/diode`); regenerated for 4.7 in 1.12.0 |
| netbox-graphql-query-optimizer | 1.x | Python | 4.0+ |
