# Performance Optimization

Detailed reference for NetBox API performance tuning.

## Critical: Config Context on Device/VM Lists

**The single most impactful optimization on NetBox 4.5–4.6 — and a no-op on 4.7+.**

| Version | Behavior | Do this |
|---------|----------|---------|
| 4.5–4.6 | Rendered per object on every request: hierarchy walk, rule evaluation at each level, merge, JSON serialization | `?exclude=config_context` on every device/VM list (or `?omit=config_context`, 4.5.2+) |
| 4.7+ | Pre-rendered and cached on the object, invalidated and re-rendered by a background job; always included; `?exclude=config_context` **silently ignored** | Nothing for speed. Use `?fields=`, `?brief=True`, or `?omit=config_context` to cut payload size |

```python
# 4.5–4.6: 10-100x slower without the exclude
requests.get(f"{API_URL}/dcim/devices/", headers=headers)
requests.get(f"{API_URL}/dcim/devices/?exclude=config_context", headers=headers)

# 4.5.2 through 4.7: one form that skips rendering on 4.5/4.6 and trims the payload on 4.7
requests.get(f"{API_URL}/dcim/devices/?omit=config_context", headers=headers)
```

Measured on 4.5–4.6:

| Devices | With config_context | Without |
|---------|-------------------|---------|
| 100 | 2-5s | 0.1-0.2s |
| 1,000 | 20-60s | 0.5-1s |
| 5,000 | Timeout likely | 2-5s |

Also applies to virtual machines.

**When config context IS needed:** on 4.5–4.6 fetch individual objects (`/dcim/devices/123/`) or small batches; on 4.7+ read it straight from the list response. In the brief window after a config context changes, 4.7 falls back to on-demand rendering for the affected objects, so the data is correct rather than stale.

## Brief Mode

`?brief=True` reduces response size by ~90%:

| Object | Full | Brief | Reduction |
|--------|------|-------|-----------|
| Device | ~2KB | ~200B | 90% |
| Prefix | ~800B | ~150B | 81% |
| Site | ~1.2KB | ~180B | 85% |

Brief returns: `id`, `url`, `display`, and natural key fields.

## Maximum Optimization

Combine parameters for maximum performance. Brief mode already drops `config_context` on every version, so `?brief=True` alone covers it:

```python
requests.get(f"{API_URL}/dcim/devices/?brief=True&limit=100", headers=headers)

# Need more than brief fields? ?fields= returns only what you list, so config_context stays out
requests.get(f"{API_URL}/dcim/devices/?fields=id,name,status,site.name&limit=100", headers=headers)
```

## Parallel Requests

Parallelize independent requests:

```python
import asyncio, httpx

async def fetch_inventory():
    async with httpx.AsyncClient(headers=headers, timeout=30) as client:
        tasks = [
            client.get(f"{API_URL}/dcim/devices/?limit=100&omit=config_context"),
            client.get(f"{API_URL}/dcim/sites/?limit=100&brief=True"),
            client.get(f"{API_URL}/ipam/prefixes/?limit=100"),
            client.get(f"{API_URL}/ipam/ip-addresses/?limit=100"),
        ]
        responses = await asyncio.gather(*tasks)
        return {
            "devices": responses[0].json()["results"],
            "sites": responses[1].json()["results"],
            "prefixes": responses[2].json()["results"],
            "ip_addresses": responses[3].json()["results"],
        }
```

## Caching Strategies

**Cache these** (change rarely): site/location hierarchy, device types and roles, tags and custom field definitions.

**Don't cache**: device status, IP address assignments, object counts.

```python
import time

class NetBoxCache:
    def __init__(self, default_ttl=300):
        self._cache = {}
        self._default_ttl = default_ttl

    def get(self, key):
        if key in self._cache:
            value, expiry = self._cache[key]
            if time.time() < expiry:
                return value
            del self._cache[key]
        return None

    def set(self, key, value, ttl=None):
        self._cache[key] = (value, time.time() + (ttl or self._default_ttl))
```

## Pagination Strategy

| Scenario | Recommended Limit |
|----------|------------------|
| Interactive UI | 25-50 |
| Background sync | 100-250 |
| Bulk export | 500-1000 |
| Streaming processing | 100 |

Larger pages reduce HTTP overhead but increase memory usage and response latency.

## Avoid `?q=` at Scale

The generic search filter becomes extremely slow with large datasets, especially devices with primary IPs. Use specific filters instead (see [rest-api-patterns.md](rest-api-patterns.md)).

## Infrastructure Considerations

| Component | Impact |
|-----------|--------|
| Database indexes | Critical — missing indexes cause severe slowdowns |
| Redis/Valkey cache | High — proper configuration dramatically impacts performance |
| Connection pooling | Medium — important for high-volume applications |
| Database maintenance | Medium — regular VACUUM and REINDEX |

## Version Performance Notes

- **v4.7.0**: config context pre-rendered and cached — device/VM lists no longer pay the render cost; hierarchies moved to PostgreSQL `ltree` and denormalized fields to triggers; global search index updates deferred to a background job (UI search may lag a write briefly)
- **v4.6.x**: cursor pagination for REST (`?start=`); N+1 fixes in GraphQL for tags, cable terminations, generic relations; faster bulk deletes (4.6.7)
- **v4.4.9+**: Includes fixes for several performance issues
- **v4.0.0**: Some performance regressions
- Always test performance before upgrading with production-like data

## Troubleshooting

### Debug Request Timing

```python
import time

def timed_request(session, url):
    start = time.time()
    response = session.get(url)
    elapsed = time.time() - start
    print(f"URL: {url}, Status: {response.status_code}, Time: {elapsed:.2f}s, Size: {len(response.content)}B")
    return response

# Compare with and without config_context — a large gap on 4.5–4.6; on 4.7+ only the payload size differs
timed_request(session, f"{API_URL}/dcim/devices/?limit=100")
timed_request(session, f"{API_URL}/dcim/devices/?limit=100&omit=config_context")
```

### Request Correlation

```python
import uuid
headers["X-Request-ID"] = str(uuid.uuid4())
# Check NetBox logs for this request ID
```

### GraphQL Debug

```python
def debug_graphql(netbox_url, token, query):
    start = time.time()
    result = graphql_query(netbox_url, token, query)
    elapsed = time.time() - start
    print(f"Time: {elapsed:.2f}s")
    if "errors" in result:
        for error in result["errors"]:
            print(f"Error: {error.get('message')}")
    return result
```

### Common Issues Checklist

1. **Slow device lists?** → On 4.5–4.6 add `?exclude=config_context`. On 4.7+ it is ignored (context is cached) — look at page size, `?q=`, and missing filters instead
2. **Large payloads?** → Use `?brief=True` or `?fields=`
3. **Slow search?** → Replace `?q=` with specific filters
4. **GraphQL timeout?** → Check pagination, depth, fan-out
5. **401 errors?** → Check token format (v1 `Token` vs v2 `Bearer`)
6. **403 errors?** → Check permissions, IP restrictions
