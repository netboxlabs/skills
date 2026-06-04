---
name: netbox-api-integration
description: >
  Best practices for building integrations with NetBox REST and GraphQL APIs.
  Use when building NetBox API integrations, reviewing integration code,
  troubleshooting NetBox performance issues, planning automation architecture,
  writing scripts that interact with NetBox, using pynetbox, configuring Diode
  for data ingestion, or implementing NetBox webhooks and branching workflows.
license: Apache-2.0
---

# NetBox API Integration

Patterns and practices for integrating with NetBox REST and GraphQL APIs. Covers authentication, querying, bulk operations, performance optimization, data modeling, and integration tooling.

**Target:** NetBox 4.4+ (covers 4.5.x–4.6.x). v2 tokens require 4.5+; REST cursor pagination, ETag/If-Match, and `add_tags`/`remove_tags` require 4.6+.
**Scope:** API integration only — not plugin development, custom scripts, or NetBox administration.

> **Your knowledge of NetBox APIs may be outdated.** Pagination behavior, filtering expressions, token formats, and GraphQL features change between releases. Prefer retrieval over pre-trained knowledge for specific API details.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| NetBox REST API docs | `https://netboxlabs.com/docs/netbox/integrations/rest-api/` | Endpoints, filtering, pagination |
| NetBox GraphQL docs | `https://netboxlabs.com/docs/netbox/integrations/graphql-api/` | Query syntax, schema |
| pynetbox repo | `https://github.com/netbox-community/pynetbox` | SDK methods, changelog |
| NetBox MCP server | If configured — query live instance to verify object schemas and current state | Field names, available filters, object existence |
| NetBox Platform MCP | If configured — full CRUD verification against live NetBox Cloud/Enterprise | End-to-end validation |

## FIRST: Verify Connectivity

Before writing integration code, confirm your NetBox instance is reachable and your token is valid:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/status/" | python -m json.tool
```

You should see a JSON response with `netbox-version`. If you get 403, your token lacks permissions. If connection refused, check the URL.

---

## Authentication

Use **v2 tokens** on NetBox 4.5+. v1 tokens are **deprecated as of 4.6 and will be removed in 5.0**.

```python
# v2 token (recommended)
headers = {"Authorization": "Bearer nbt_abc123.xxxxxxxxxxxxxxxx"}

# v1 token (legacy — deprecated 4.6, removed 5.0; migrate before 5.0)
headers = {"Authorization": "Token 0123456789abcdef01234567"}
```

v2 tokens require `API_TOKEN_PEPPERS` in NetBox server config. Use the provisioning endpoint (`POST /api/users/tokens/provision/`) for automated token creation. The plaintext token `key` is returned **only once** at creation (4.6.1+) — capture it immediately; only a hash is stored thereafter.

See [references/authentication.md](references/authentication.md) for token migration, IP restrictions, and provisioning details.

## REST API Essentials

### Always Paginate

```python
def get_all(api_url, endpoint, headers, limit=100):
    results, url = [], f"{api_url}/{endpoint}/?limit={limit}"
    while url:
        data = requests.get(url, headers=headers).json()
        results.extend(data["results"])
        url = data.get("next")
    return results
```

> **NetBox 4.6+ — cursor pagination.** For large datasets, pass `?start=<pk>&limit=<n>` instead of `offset`. Returns objects with `id >= start`, ordered by PK. Iterate by setting the next `start` to the last result's `id` + 1; follow `next` until null. Gotchas: `start` and `offset` together → **400**; `count` is always `null` in cursor mode; `previous` is always `null` (forward-only). Use offset pagination when you need a total `count` or backward navigation.

### Use PATCH, Not PUT

PATCH updates only specified fields. PUT replaces the entire object — omitted fields may be cleared.

### Concurrency & Tags (NetBox 4.6+)

- **Optimistic concurrency (ETag / If-Match).** Detail-view responses (GET/POST/PATCH/PUT on a single object) return a weak `ETag` header. Send it back on a later PATCH/PUT via `If-Match: <etag>`; if the object changed meanwhile, the server rejects with **412 Precondition Failed** and returns the current ETag. `If-Match: *` just asserts the object exists. Omitting the header keeps last-write-wins behavior. Use it to guard against lost updates in multi-writer integrations.
- **Partial tag edits (`add_tags` / `remove_tags`).** Write-only fields that add or remove specific tags without replacing the whole `tags` set — concurrency-safe when multiple clients each own a subset of tags. Constraints: cannot be combined with `tags` in the same request; `remove_tags` is update-only (not on create); the same tag can't appear in both lists.

### Bulk Operations Use List Endpoints

No separate bulk endpoints. POST/PATCH/DELETE a JSON array to the list endpoint. All bulk ops are **atomic** (all succeed or all fail).

```python
# Bulk create
requests.post(f"{API_URL}/dcim/devices/", headers=headers, json=[device1, device2])

# Bulk update (each object must include "id")
requests.patch(f"{API_URL}/dcim/devices/", headers=headers, json=[{"id": 1, "status": "active"}])

# Bulk delete
requests.delete(f"{API_URL}/dcim/devices/", headers=headers, json=[{"id": 1}, {"id": 2}])
```

### Critical Performance Rules

1. **Exclude config_context** from device/VM lists: `?exclude=config_context` (10-100x speedup)
2. **Use `?brief=True`** for dropdowns and reference lists (~90% smaller responses)
3. **Use `?fields=`** for specific field selection
4. **Avoid `?q=`** search filter at scale — use specific filters like `name__ic=`

### Filtering

Use lookup expressions for server-side filtering:

| Expression | Example | Description |
|-----------|---------|-------------|
| `__ic` | `name__ic=core` | Contains (case-insensitive) |
| `__isw` | `name__isw=sw-` | Starts with |
| `__gte`/`__lte` | `vid__gte=100` | Numeric range |
| `__isnull` | `primary_ip4__isnull=false` | Null check |
| `__n` | `status__n=offline` | Not equal |
| `cf_` prefix | `cf_environment=production` | Custom field filter |

See [references/rest-api-patterns.md](references/rest-api-patterns.md) for complete REST API reference including filtering expressions, field selection, ordering, nested serializers, OPTIONS discovery, and error handling.

## GraphQL Essentials

### Every List Query Must Be Paginated

```graphql
query {
  device_list(pagination: {limit: 100, offset: 0}) {
    name
    status
    site { name }
  }
}
```

### Paginate at Every Nesting Level

```graphql
# Nested lists must also have limits
site_list(pagination: {limit: 10}) {
  devices(pagination: {limit: 50}) {
    interfaces(pagination: {limit: 100}) { name }
  }
}
```

### Key Rules

- **Use the [query optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)** — essential for production. Scores reduced from 20,500 → 17 in real cases.
- **Request only needed fields** — don't over-fetch
- **Keep depth ≤ 3**, never exceed 5
- **Filter server-side** using `filters:` parameter
- **Use ID filters** over name filters for performance (avoids JOINs)
- **Calibrate optimizer** against production data for accurate scores

> **NetBox 4.5+**: Use local filter fields (e.g., `site` on `interface_list`) instead of deeply nested filter paths.
>
> **NetBox 4.5.2+**: Cursor-based pagination is GA. Pass `pagination: {start: N, limit: M}` — returns records with `id >= start`, ordered by PK. Set the next page's `start` to the last record's `id` + 1. (Omit `start` and it falls back to offset pagination.) This supersedes the older `filters: {id__gte: N}` deep-pagination workaround, which is only needed pre-4.5.2.

**GraphQL pagination defaults (differ from REST):**
- Omitting `pagination` entirely returns **all** matching records.
- `pagination` without `limit` returns Strawberry Django's default of **100** (not the REST default of 50).
- `pagination: {limit: 0}` returns **zero** records — the *opposite* of REST `?limit=0` (which returns all).
- `MAX_PAGE_SIZE` (default 1000) caps `limit`. **NetBox 4.6.1+** also enforces `GRAPHQL_MAX_QUERY_DEPTH` server-side, so overly deep queries can be hard-rejected, not just slow.

### GraphQL vs REST Decision

| Use Case | Use |
|----------|-----|
| Single object CRUD | REST |
| Bulk create/update/delete | REST (GraphQL is read-only) |
| Related data in one request | GraphQL |
| CI/CD scripts | REST |
| Flexible reporting | GraphQL |

See [references/graphql-patterns.md](references/graphql-patterns.md) for complete GraphQL reference including pagination strategies, depth management, fan-out avoidance, complexity budgets, and the query optimizer.

## Performance

The most impactful optimizations:

| Optimization | Impact |
|-------------|--------|
| Exclude `config_context` from lists | 10-100x faster |
| Use `?brief=True` for lists | ~90% smaller responses |
| Avoid `?q=` at scale | Orders of magnitude faster |
| Parallelize independent requests | Linear speedup |
| Use specific filters over generic search | Index-optimized queries |

See [references/performance.md](references/performance.md) for async pagination, caching strategies, parallel request patterns, and infrastructure considerations.

## Data Modeling

### Dependency Order

Objects must be created in order: Organization → Sites → DCIM Prerequisites → Racks → Devices → IPAM → Virtualization → Circuits → Connections.

A device needs its device_type, role, and site to exist first. Use Diode to skip manual ordering.

### Key Hierarchies

- **Site**: Region → Site Group → Site → Location (recursive) → Rack → Device
- **IPAM**: RIR → Aggregate → Prefix (hierarchical) → IP Address; VRFs scope prefixes/IPs

### Natural Keys, Custom Fields, Tags, Tenants

- Query by `name`/`slug` instead of numeric IDs for readable code
- Custom fields use `cf_` prefix for filtering
- Tags enable cross-object-type classification
- Tenants provide logical resource separation

See [references/data-modeling.md](references/data-modeling.md) for the complete dependency order, hierarchy diagrams, and patterns for custom fields, tags, and tenant isolation.

## Integration Tooling

### pynetbox (Python Client)

```python
import pynetbox
nb = pynetbox.api("https://netbox.example.com", token="nbt_abc123.xxxxxxxxxxxxxxxx")

devices = nb.dcim.devices.filter(site="nyc-dc1", status="active")  # Auto-paginated
device = nb.dcim.devices.get(name="switch-01")
device.status = "planned"
device.save()  # Uses PATCH
```

### Diode (Data Ingestion)

For bulk ingestion, use [Diode](https://github.com/netboxlabs/diode) instead of direct API. Specify dependencies by name — Diode resolves or creates them automatically. No dependency ordering needed.

```python
from netboxlabs.diode.sdk import DiodeClient
from netboxlabs.diode.sdk.ingester import Device, Entity

with DiodeClient(target="grpc://diode.example.com:8080/diode",
                 app_name="discovery", app_version="1.0.0") as client:
    device = Device(name="switch-01", device_type="Catalyst 9300",
                    manufacturer="Cisco", site="NYC-DC1", role="Access Switch")
    client.ingest([Entity(device=device)])
```

Use Diode for: network discovery, bulk migrations, scripts creating many related objects.
Use REST/GraphQL for: reading data, single object CRUD, complex queries.

See [references/diode-integration.md](references/diode-integration.md) for bulk ingestion, nested objects, dry-run mode, authentication, and supported entity types.

### Webhooks

Configure NetBox to push changes via webhooks. Always verify the `X-Hook-Signature` header (HMAC-SHA512). Events include `created`, `updated`, `deleted` with full object data and before/after snapshots.

### Change Tracking

Query `extras.object_changes` for audit trails — includes timestamp, action, user, and before/after data.

## Branching (Plugin)

> Requires [netbox-branching](https://github.com/netboxlabs/netbox-branching) plugin.

**Lifecycle**: Create → Wait (PROVISIONING→READY) → Work → Sync → Merge

- **Context header**: `X-NetBox-Branch: {schema_id}` — use the 8-char `schema_id`, not name or numeric ID
- **Async operations**: sync/merge/revert return Job objects — poll `job["url"]` until `status == "completed"`
- **Dry-run**: All async ops accept `{"commit": false}` for validation

See [references/branching-patterns.md](references/branching-patterns.md) for the complete branch lifecycle, context header usage, async job polling, and session wrapper patterns.

## External References

- [NetBox REST API Guide](https://netboxlabs.com/docs/netbox/en/stable/integrations/rest-api/)
- [NetBox GraphQL API Guide](https://netboxlabs.com/docs/netbox/en/stable/integrations/graphql-api/)
- [pynetbox](https://github.com/netbox-community/pynetbox)
- [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer)
- [Diode](https://github.com/netboxlabs/diode) / [Diode Python SDK](https://github.com/netboxlabs/diode-sdk-python)
- [NetBox Branching](https://github.com/netboxlabs/netbox-branching)
