# Performance Optimization

Detailed reference for NetBox API performance tuning.

## Critical: Exclude Config Context

**The single most impactful optimization for device/VM queries.**

```python
# SLOW: 10-100x slower
requests.get(f"{API_URL}/dcim/devices/", headers=headers)

# FAST
requests.get(f"{API_URL}/dcim/devices/?exclude=config_context", headers=headers)
```

Config context computation involves traversing the device hierarchy, evaluating context rules at each level, merging multiple sources, and serializing potentially large JSON.

| Devices | With config_context | Without |
|---------|-------------------|---------|
| 100 | 2-5s | 0.1-0.2s |
| 1,000 | 20-60s | 0.5-1s |
| 5,000 | Timeout likely | 2-5s |

Also applies to virtual machines: `?exclude=config_context`.

**When config context IS needed:** Fetch individual objects (`/dcim/devices/123/`) or specific small batches.

## Brief Mode

`?brief=True` reduces response size by ~90%:

| Object | Full | Brief | Reduction |
|--------|------|-------|-----------|
| Device | ~2KB | ~200B | 90% |
| Prefix | ~800B | ~150B | 81% |
| Site | ~1.2KB | ~180B | 85% |

Brief returns: `id`, `url`, `display`, and natural key fields.

## Maximum Optimization

Combine all parameters for maximum performance:

```python
requests.get(
    f"{API_URL}/dcim/devices/?exclude=config_context&brief=True&limit=100",
    headers=headers)
```

## Parallel Requests

Parallelize independent requests:

```python
import asyncio, httpx

async def fetch_inventory():
    async with httpx.AsyncClient(headers=headers, timeout=30) as client:
        tasks = [
            client.get(f"{API_URL}/dcim/devices/?limit=100&exclude=config_context"),
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

- **v4.0.0**: Some performance regressions
- **v4.4.9+**: Includes fixes for several performance issues
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

# Compare with and without config_context
timed_request(session, f"{API_URL}/dcim/devices/?limit=100")
timed_request(session, f"{API_URL}/dcim/devices/?limit=100&exclude=config_context")
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

1. **Slow device lists?** → Add `?exclude=config_context`
2. **Large payloads?** → Use `?brief=True` or `?fields=`
3. **Slow search?** → Replace `?q=` with specific filters
4. **GraphQL timeout?** → Check pagination, depth, fan-out
5. **401 errors?** → Check token format (v1 `Token` vs v2 `Bearer`)
6. **403 errors?** → Check permissions, IP restrictions
