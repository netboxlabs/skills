# Code-Mode Workflows

Illustrative programs to adapt. Each is meant to be **one** `netbox_execute` call, after at most **one** `netbox_search_schema` call. Always assign `result`; never `import`; project with `fields=`.

## A. Active devices at a site, projected

```python
sites = get('dcim.site', filters={'name': 'NYC-DC1'}, fields=['id', 'name'])
site_id = sites['results'][0]['id']
devs = get('dcim.device', filters={'site_id': site_id, 'status': 'active'},
           fields=['id', 'name', 'role', 'primary_ip4'], limit=1000)
result = {'site': 'NYC-DC1', 'count': devs['count'], 'devices': devs['results']}
```

## B. Filter by a list of IDs (correct `__in` form)

```python
result = get('dcim.interface', filters={'id': [621493, 631527, 644001]},
             fields=['id', 'name', 'device'])
```

## C. Two-step cross-relationship query (no multi-hop)

```python
racks = get('dcim.rack', filters={'site': 'nyc-dc1'}, fields=['id'])
rack_ids = [r['id'] for r in racks['results']]
devices = get('dcim.device', filters={'rack_id': rack_ids}, fields=['id', 'name', 'rack'])
result = devices['results']
```

## D. Allocate the next free /30 from a parent prefix (write + IPAM gates)

```python
parents = get('ipam.prefix', filters={'prefix': '10.0.0.0/16'}, fields=['id'])
new_prefix = allocate_prefix(parents['results'][0]['id'], 30,
                             description='p2p link', status='active')
result = new_prefix
```

## E. Audit recent deletions (changelog; imperative-singular action)

```python
result = changelogs(filters={'action': 'delete', 'time_after': '2026-05-01T00:00:00Z'},
                    limit=100)
```

## F. Schema discovery before acting (search tool)

```python
# netbox_search_schema — find the right object type before guessing
result = {k: v for k, v in schema['object_types'].items() if 'vlan' in k.lower()}
```

## G. Inspect required fields before a create (avoid a failed write)

```python
result = options('dcim.device')   # required fields, choices — confirm before create()
```

## H. Loop a computed report in one program (don't call once per object)

```python
prefixes = get('ipam.prefix', filters={'status': 'active'},
               fields=['id', 'prefix', 'description'], limit=1000)
rows = []
for p in prefixes['results']:
    free = available_ips(p['id'], limit=1)
    rows.append({'prefix': p['prefix'], 'has_free': len(free) > 0})
result = {'count': len(rows), 'prefixes': rows}
```

## Recovery — only when the tool inventory is genuinely stale

```python
# A plugin was just installed/uninstalled and you need the new tools/helpers visible.
# Call the bare tool — do NOT use this to "refresh data."
rediscover_netbox()
```

## Handling a write error

HTTP errors surface the NetBox response body, including per-field validation detail. On a failed `create()`/`update()`, read the returned error, correct the payload, and retry in the next program — don't blindly repeat the same call.
