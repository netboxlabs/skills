# GraphQL Patterns

Complete reference for NetBox GraphQL API usage, query optimization, and performance.

## Endpoint

```
POST https://netbox.example.com/graphql/
```

All queries use POST with `{"query": "...", "variables": {...}}`.

## Basic Query

```python
import requests

def graphql_query(netbox_url, token, query, variables=None):
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    payload = {"query": query}
    if variables:
        payload["variables"] = variables
    response = requests.post(f"{netbox_url}/graphql/", headers=headers, json=payload)
    response.raise_for_status()
    result = response.json()
    if "errors" in result:
        raise Exception(f"GraphQL errors: {result['errors']}")
    return result["data"]
```

## Pagination

### Every List Must Be Paginated

```graphql
# WRONG — unbounded, could return entire database
query { device_list { name } }

# CORRECT
query { device_list(pagination: {limit: 100, offset: 0}) { name } }
```

### Paginate at Every Nesting Level

```graphql
# WRONG — nested lists unbounded: 10 × unlimited × unlimited
site_list(pagination: {limit: 10}) {
  devices { interfaces { name } }
}

# CORRECT — bounded at every level
site_list(pagination: {limit: 10}) {
  devices(pagination: {limit: 50}) {
    interfaces(pagination: {limit: 100}) { name }
  }
}
```

### Full Pagination Implementation

```python
def fetch_all_graphql(netbox_url, token, query, list_key, page_size=100):
    all_items, offset = [], 0
    while True:
        data = graphql_query(netbox_url, token, query, {"limit": page_size, "offset": offset})
        items = data[list_key]
        if not items:
            break
        all_items.extend(items)
        if len(items) < page_size:
            break
        offset += page_size
    return all_items
```

### Offset Pagination Performance at Scale

Offset pagination scans all rows up to the offset — performance degrades linearly:

| Offset | Rows Scanned | Typical Latency |
|--------|--------------|-----------------|
| 0 | 100 | ~50ms |
| 10,000 | 10,100 | ~500ms |
| 100,000 | 100,100 | ~5s or timeout |

**Version-specific solutions:**

| Version | Approach |
|---------|----------|
| ≤ 4.5.1 | Offset only, or the ID-range filtering workaround below |
| 4.5.2+ | **Cursor-based pagination** — `pagination: {start: N, limit: M}` (GA) |

**Cursor pagination (4.5.2+, preferred):**

```graphql
query GetDevices($start: Int!, $limit: Int!) {
  device_list(pagination: {start: $start, limit: $limit}) {
    id
    name
  }
}
```

Returns records with `id >= start`, always ordered by PK. For the next page, set `start` to the last record's `id` + 1. Omitting `start` falls back to offset pagination. This supersedes the ID-range workaround below (only needed pre-4.5.2).

**ID range pagination (≤ 4.5.1 fallback):**

```graphql
query GetDevicesAfter($lastId: ID!, $limit: Int!) {
  device_list(pagination: {limit: $limit}, filters: {id: {gte: $lastId}}) {
    id
    name
  }
}
```

```python
def fetch_with_id_cursor(netbox_url, token, page_size=100):
    all_results, min_id = [], 0
    while True:
        data = graphql_query(netbox_url, token, query, {"lastId": min_id, "limit": page_size})
        items = data["device_list"]
        if not items:
            break
        all_results.extend(items)
        if len(items) < page_size:
            break
        min_id = max(item["id"] for item in items) + 1
    return all_results
```

Caveats: inconsistent page sizes with ID gaps, must fetch `id` in every query, cannot jump to arbitrary pages. On 4.5.2+ prefer the native `start` cursor above.

### GraphQL Pagination Defaults (differ from REST)

- Omitting `pagination` entirely returns **all** matching records.
- `pagination` without `limit` returns Strawberry Django's default of **100** (the REST default of 50 does **not** apply to GraphQL).
- `pagination: {limit: 0}` returns **zero** records — the opposite of REST `?limit=0` (which returns all).
- `MAX_PAGE_SIZE` (default 1000) caps `limit`.

## Field Selection

Only request fields your application uses:

```graphql
# WRONG — over-fetching
query { device_list(pagination: {limit: 100}) { id name status device_type { id model slug manufacturer { id name slug description } } role { id name slug color } ... } }

# CORRECT — minimal fields
query { device_list(pagination: {limit: 100}) { name status device_type { model } } }
```

Each unnecessary nested object adds JOINs and serialization overhead.

## Query Depth

Keep depth ≤ 3. Never exceed 5. Each level multiplies complexity. **NetBox 4.6.1+** enforces `GRAPHQL_MAX_QUERY_DEPTH` server-side, so an over-deep query can be hard-rejected (not merely slow) depending on instance config.

```
Level 1: site_list
├── Level 2: devices
│   ├── Level 3: interfaces (max recommended)
│   │   └── Level 4: ip_addresses (AVOID)
│   │       └── Level 5: vrf (NEVER)
```

**Split deep queries** into multiple shallow ones:

```graphql
# Query 1: Sites with devices (depth 2)
query { site_list(pagination: {limit: 10}) { id name devices(pagination: {limit: 20}) { id name } } }

# Query 2: Interfaces for specific device (depth 2)
query($deviceId: ID!) { interface_list(filters: {device_id: $deviceId}, pagination: {limit: 100}) { name ip_addresses(pagination: {limit: 10}) { address } } }
```

## Server-Side Filtering

> **Filter type rules (4.5+):**
> - Foreign key ID fields (`site_id`, `device_id`, etc.) are **scalar `ID`** type — use `filters: {site_id: $id}` (no `{exact:}` wrapper). Declare variables as `$id: ID!`.
> - String/text fields use lookup objects: `filters: {name: {exact: "value"}}`.
> - Status/choice fields use **enum values** (unquoted, prefixed): `status: {exact: STATUS_ACTIVE}`, not `"active"`.
> - Relationship filters are nested objects: `site: {name: {exact: "NYC-DC1"}}`.

Always filter in the query, not client-side:

```graphql
# WRONG: Fetch 1000, filter to 50 client-side
query { device_list(pagination: {limit: 1000}) { name status site { name } } }

# CORRECT: Filter server-side
query { device_list(pagination: {limit: 100}, filters: {status: {exact: STATUS_ACTIVE}, site: {name: {exact: "NYC-DC1"}}}) { name status } }
```

### Filter by ID for Performance

Use numeric IDs instead of names to avoid SQL JOINs:

```graphql
# SLOWER: Name filter requires JOIN to sites table
filters: {site: {name: {exact: "NYC-DC1"}}}

# FASTER: ID filter uses local foreign key column (scalar, no {exact:} wrapper)
filters: {site_id: $siteId}  # where $siteId: ID!
```

Cache ID lookups for repeated queries.

### Avoid Deeply Nested Filters

> **NetBox 4.5.1+**: Use local filter fields instead of traversing relationships.

```graphql
# SUBOPTIMAL: Filter depth 3
filters: { device: { site: { name: {exact: "NYC-DC1"} } } }

# OPTIMAL: Local filter (4.5.1+)
filters: { site: {name: {exact: "NYC-DC1"}} }
```

Common local filters: `interface_list` has `site`, `device_role`; `ip_address_list` has `site`; `cable_list` has `site`. Use GraphQL introspection to discover available filters.

## The Query Optimizer

The [netbox-graphql-query-optimizer](https://github.com/netboxlabs/netbox-graphql-query-optimizer) is essential for production GraphQL. It detects N+1 queries, unbounded lists, fan-out patterns, and depth violations.

**Real impact:** Scores reduced from 20,500 → 17 (~1,200x improvement).

```bash
pip install netbox-graphql-query-optimizer

# Basic analysis
netbox-query-optimizer analyze query.graphql

# Calibrated against real data (recommended for production)
netbox-query-optimizer analyze query.graphql \
  --calibrate --url https://netbox.example.com --token nbt_abc123.xxxxx
```

**Always calibrate against production.** Default scores are estimates. Your NetBox with 5,000 devices will have very different scores than the defaults.

### Complexity Budgets

| Query Type | Max Score |
|------------|-----------|
| Dashboard widgets | 50 |
| Autocomplete | 25 |
| List views | 150 |
| Detail views | 200 |
| Reports | 500 |
| ETL/sync jobs | 1000 |

### CI Integration

```yaml
- run: |
    for query in queries/*.graphql; do
      netbox-query-optimizer analyze "$query" --max-score 500 --fail-on-warning
    done
```

### Calibration Caching

```bash
netbox-query-optimizer calibrate --url $URL --token $TOKEN --output calibration.json
netbox-query-optimizer analyze query.graphql --calibration-file calibration.json
```

Recalibrate on significant data growth, distribution changes, or quarterly.

## Fan-Out Avoidance

Fan-out multiplies objects: `10 sites × 100 devices × 50 interfaces = 50,000 objects`.

Instead of deep fan-out queries, use summary first, then drill down:

```graphql
# Step 1: Summary — get site IDs
query { site_list(pagination: {limit: 10}) { id name } }

# Step 2: Detail for specific site (site_id is scalar ID, no {exact:} wrapper)
query($siteId: ID!) { device_list(filters: {site_id: $siteId}, pagination: {limit: 100}) { name interface_count } }
```

## GraphQL vs REST Decision

| Use Case | Use |
|----------|-----|
| Single object by ID | REST |
| Simple filtered list | REST |
| Multiple related types in one request | GraphQL |
| Bulk create/update/delete | REST (GraphQL is read-only) |
| Real-time dashboards | REST (easier HTTP caching) |
| Flexible reporting | GraphQL |
| CI/CD scripts | REST |

Hybrid approach: REST for writes, GraphQL for complex reads.

## Error Handling

GraphQL returns 200 even with query errors. Always check for `errors` in the response:

```python
result = response.json()
if "errors" in result:
    for error in result["errors"]:
        print(f"Error: {error.get('message')}, Location: {error.get('locations')}")
```

HTTP-level errors (401, 403) still use standard status codes.
