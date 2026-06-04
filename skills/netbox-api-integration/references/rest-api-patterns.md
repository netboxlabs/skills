# REST API Patterns

Complete reference for NetBox REST API usage patterns.

## Base URL

```
https://netbox.example.com/api/
```

## HTTP Methods

| Method | Purpose | Idempotent |
|--------|---------|------------|
| GET | Retrieve | Yes |
| POST | Create | No |
| PUT | Replace entire object | Yes |
| PATCH | Partial update | Yes |
| DELETE | Remove | Yes |
| OPTIONS | Discover schema | Yes |

## PATCH vs PUT

Always use PATCH unless you intend to replace the entire object:

```python
# PATCH: Only status changes, other fields preserved
requests.patch(f"{API_URL}/dcim/devices/123/",
    headers=headers, json={"status": "active"})

# PUT: Must send complete object — omitted fields may be cleared
requests.put(f"{API_URL}/dcim/devices/123/",
    headers=headers, json={...complete object...})
```

## Bulk Operations

Standard list endpoints accept JSON arrays. No separate bulk endpoints exist.

```python
# Bulk create (POST array)
requests.post(f"{API_URL}/dcim/devices/", headers=headers, json=[
    {"name": "sw-01", "device_type": 1, "role": 1, "site": 1},
    {"name": "sw-02", "device_type": 1, "role": 1, "site": 1},
])

# Bulk update (PATCH array — each must include "id")
requests.patch(f"{API_URL}/dcim/devices/", headers=headers, json=[
    {"id": 1, "status": "active"},
    {"id": 2, "status": "active"},
])

# Bulk delete (DELETE array)
requests.delete(f"{API_URL}/dcim/devices/", headers=headers, json=[
    {"id": 1}, {"id": 2}
])
```

**Atomicity:** All bulk operations are atomic — if any item fails validation, the entire operation rolls back.

**Signals/webhooks:** Each object in the array triggers its own Django signals and webhook events. This is intentional for proper validation; bulk operations are not true database-level bulk inserts.

## Pagination

Default: 50 items. Maximum: 1000. Always specify `limit` explicitly.

**Response format:**
```json
{
    "count": 1500,
    "next": "https://netbox.example.com/api/dcim/devices/?limit=100&offset=100",
    "previous": null,
    "results": [...]
}
```

**Complete pagination:**
```python
def get_all(api_url, endpoint, headers, limit=100):
    results, url = [], f"{api_url}/{endpoint}/?limit={limit}"
    while url:
        resp = requests.get(url, headers=headers)
        resp.raise_for_status()
        data = resp.json()
        results.extend(data["results"])
        url = data.get("next")
    return results
```

**Async parallel pagination:**
```python
import asyncio, httpx

async def get_all_async(api_url, endpoint, headers, limit=100):
    async with httpx.AsyncClient(headers=headers) as client:
        resp = await client.get(f"{api_url}/{endpoint}/?limit=1")
        total = resp.json()["count"]
        pages = (total + limit - 1) // limit
        tasks = [client.get(f"{api_url}/{endpoint}/?limit={limit}&offset={i*limit}")
                 for i in range(pages)]
        responses = await asyncio.gather(*tasks)
        return [item for r in responses for item in r.json()["results"]]
```

**Recommended page sizes:**

| Use Case | Limit |
|----------|-------|
| Interactive UI | 25-50 |
| Background sync | 100-250 |
| Bulk export | 500-1000 |

### Cursor Pagination (NetBox 4.6+)

For very large datasets, offset pagination slows down as the DB scans all rows up to the offset. Cursor (keyset) pagination filters by PK instead — pass `start` (minimum `id`) and `limit`:

```python
def get_all_cursor(api_url, endpoint, headers, limit=1000):
    results, url = [], f"{api_url}/{endpoint}/?start=0&limit={limit}"
    while url:
        resp = requests.get(url, headers=headers)
        resp.raise_for_status()
        data = resp.json()
        results.extend(data["results"])
        url = data.get("next")   # server builds next start for you
    return results
```

- Manual iteration: set the next request's `start` to the last result's `id` + 1.
- `start` and `offset` together → **400**.
- `count` is always `null` in cursor mode (counting would negate the speedup).
- `previous` is always `null` — forward-only.
- Use offset pagination when you need a total `count` or backward navigation.

## Brief Mode

`?brief=True` returns minimal fields (~90% smaller): `id`, `url`, `display`, and natural key fields.

| Object | Full | Brief | Reduction |
|--------|------|-------|-----------|
| Device | ~2KB | ~200B | 90% |
| Prefix | ~800B | ~150B | 81% |

Use for: dropdowns, autocomplete, existence checks, relationship validation.

## Field Selection

`?fields=field1,field2,nested.field` for fine-grained control:

```python
requests.get(f"{API_URL}/dcim/devices/?fields=id,name,status,site.name,primary_ip4.address",
    headers=headers)
```

Use `?fields=` when you need specific non-brief fields. Use `?brief=True` for simple dropdowns.

## Excluding Fields

`?exclude=field1,field2` omits specific fields:

```python
# CRITICAL: Exclude config_context from device/VM lists
requests.get(f"{API_URL}/dcim/devices/?exclude=config_context", headers=headers)
```

Config context is the single most expensive field — 10-100x slower with it included.

`?omit=field1,field2` (NetBox 4.5.2+) is a companion to `?fields=`/`?exclude=` that drops the named fields from full responses. `fields` and `omit` are mutually exclusive — if both are passed, `fields` wins.

## Optimistic Concurrency (ETag / If-Match, NetBox 4.6+)

Detail-view responses for a single object (GET/POST/PATCH/PUT) include a weak `ETag` header. Send it back on a later write via `If-Match` to guard against the lost-update problem:

```python
obj = requests.get(f"{API_URL}/dcim/sites/1/", headers=headers)
etag = obj.headers["ETag"]

resp = requests.patch(f"{API_URL}/dcim/sites/1/", headers={**headers, "If-Match": etag},
                      json={"status": "decommissioning"})
if resp.status_code == 412:        # object changed since we read it
    # response body carries the current ETag — re-read and retry
    ...
```

- Mismatch → **412 Precondition Failed**, with the current ETag in the response.
- `If-Match: *` asserts only that the object exists.
- Omitting `If-Match` keeps prior last-write-wins behavior.

## Partial Tag Edits (add_tags / remove_tags, NetBox 4.6+)

Taggable models accept write-only `add_tags` / `remove_tags` fields that adjust only the named tags without replacing the whole set — safer than `tags=` when multiple writers each manage a subset:

```python
requests.patch(f"{API_URL}/dcim/sites/1/", headers=headers,
    json={"add_tags": [{"name": "production"}], "remove_tags": [{"name": "staging"}]})
```

- Cannot be combined with `tags` in the same request.
- `remove_tags` is update-only (not valid on create).
- The same tag may not appear in both lists.

## Filtering

### Basic Filters

```python
requests.get(f"{API_URL}/dcim/devices/?status=active&site=nyc-dc1", headers=headers)

# Multiple values (OR logic)
requests.get(f"{API_URL}/dcim/devices/?status=active&status=planned", headers=headers)
```

### Lookup Expressions

| Suffix | Description | Example |
|--------|-------------|---------|
| (none) | Exact match | `name=switch-01` |
| `__n` | Not equal | `status__n=offline` |
| `__ic` | Contains (case-insensitive) | `name__ic=core` |
| `__nic` | Not contains | `name__nic=test` |
| `__isw` | Starts with | `name__isw=sw-` |
| `__nisw` | Not starts with | `name__nisw=temp-` |
| `__iew` | Ends with | `name__iew=-prod` |
| `__niew` | Not ends with | `name__niew=-dev` |
| `__ie` | Exact (case-insensitive) | `name__ie=Switch-01` |
| `__nie` | Not exact (case-insensitive) | `name__nie=Router-01` |
| `__empty` | Is empty | `description__empty=true` |
| `__gte` | Greater than or equal | `vlan_id__gte=100` |
| `__gt` | Greater than | `vlan_id__gt=100` |
| `__lte` | Less than or equal | `vlan_id__lte=200` |
| `__lt` | Less than | `vlan_id__lt=200` |
| `__isnull` | Is null | `primary_ip4__isnull=false` |

### Custom Field Filters

Use the `cf_` prefix:

```python
requests.get(f"{API_URL}/dcim/devices/?cf_environment=production&cf_tier=1", headers=headers)
requests.get(f"{API_URL}/dcim/devices/?cf_deployment_date__gte=2024-01-01", headers=headers)
```

### Avoid `q=` at Scale

The `?q=` search parameter searches multiple fields simultaneously without index optimization. It becomes extremely slow with large datasets, especially devices with primary IPs.

| Query | 100 devices | 5000 devices | 20000 devices |
|-------|-------------|--------------|---------------|
| `q=switch` | 0.5s | 10-30s | 60s+ |
| `name__ic=switch` | 0.1s | 0.3s | 0.8s |

Use specific filters instead. For multi-field search, run parallel queries against specific fields and deduplicate results.

`?q=` is acceptable for: small datasets (<500 objects), one-off manual queries, admin/debugging.

## Ordering

```python
requests.get(f"{API_URL}/dcim/devices/?ordering=name", headers=headers)       # Ascending
requests.get(f"{API_URL}/dcim/devices/?ordering=-created", headers=headers)    # Descending
requests.get(f"{API_URL}/dcim/devices/?ordering=site,name", headers=headers)   # Multiple
```

## Nested Serializers

Responses include nested representations for related objects. When creating/updating, use integer IDs:

```python
# Response contains nested objects
device = {"name": "sw-01", "site": {"id": 1, "name": "NYC-DC1", "slug": "nyc-dc1"}}

# But create/update uses integer IDs
requests.post(f"{API_URL}/dcim/devices/", headers=headers,
    json={"name": "sw-01", "site": 1, "device_type": 1, "role": 1})
```

## OPTIONS Discovery

```python
schema = requests.options(f"{API_URL}/dcim/devices/", headers=headers).json()
for field, meta in schema["actions"]["POST"].items():
    print(f"{field}: {meta.get('type')}, required={meta.get('required', False)}")
```

## Request Correlation

Use `X-Request-ID` header to correlate requests with NetBox logs:

```python
import uuid
headers["X-Request-ID"] = str(uuid.uuid4())
```

## Error Handling

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success (GET/PATCH/PUT) | Process data |
| 201 | Created (POST) | Process data |
| 204 | No Content (DELETE) | Success |
| 400 | Validation error | Fix input — response body has field-level errors |
| 401 | Unauthorized | Check token format and validity |
| 403 | Forbidden | Check permissions, IP restrictions |
| 404 | Not Found | Check endpoint/ID |
| 412 | Precondition Failed (`If-Match` ETag mismatch, 4.6+) | Re-read object, reconcile, retry with new ETag |
| 429 | Rate Limited | Backoff using `Retry-After` header |
| 500+ | Server Error | Retry with exponential backoff |

Error response format:
```json
{"name": ["This field is required."], "site": ["Invalid pk \"999\" - object does not exist."]}
```

Implement retry logic with exponential backoff for 429 and 500+ errors. Always set a request timeout (30s default).
