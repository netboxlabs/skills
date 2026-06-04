---
name: netboxlabs-platform-mcp
description: >
  Drive the NetBox Labs Platform MCP server (hosted on NetBox Cloud) from an AI agent.
  Covers connecting/auth, Code Mode (the three tools netbox_search_schema / netbox_execute
  / rediscover_netbox, the injected helper catalog, the sandbox contract, round-trip
  minimization) and Discrete per-tool mode. Use when an agent connects to the Platform MCP
  server, writes code-mode programs against NetBox, or chooses between code and discrete mode.
license: Apache-2.0
---

# NetBox Labs Platform MCP Server

The **Platform MCP Server** is a hosted Model Context Protocol server (on NetBox Cloud) that puts a NetBox instance behind an MCP tool surface so an agent can read and modify NetBox data. It is distinct from the open-source [`netbox-community/netbox-mcp-server`](https://github.com/netbox-community/netbox-mcp-server): the platform server adds **Code Mode**, per-user auth, dynamic plugin/model detection, and first-party product integrations (Assurance, Discovery).

> **Your knowledge of this server may be outdated.** It is Public Preview; tool inventories, helper signatures, and available namespaces are generated live per deployment and evolve. Always discover the real surface (schema search / tool list) before acting — never assume an object type, endpoint, or helper exists.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Platform MCP docs | `https://netboxlabs.com/docs/cloud/platform-mcp-server` | Setup, client configs, auth, modes |
| Community MCP server | `https://github.com/netbox-community/netbox-mcp-server` | The open-source read-only alternative |
| Live server | The connected MCP server itself | The authoritative tool list + `netbox_search_schema` are the source of truth |

## Connect & Auth

- **Endpoint:** `https://<instance>.cloud.netboxapp.com/mcp`
- **Transport:** MCP over **streamable HTTP** (no stdio — it is a networked, hosted server, not a local subprocess).
- **Auth:** `Authorization: Bearer nbt_<token>` — a **NetBox v2 API token** (prefix `nbt_`, available on NetBox 4.5+). With per-user auth, your own token's NetBox RBAC applies to every call.
- Customers enable the server on their instance via NetBox Labs support.

Any MCP-over-streamable-HTTP client works (Claude Code/Desktop, Cursor, VS Code, ChatGPT Developer Mode). The docs page has copy-paste client configs.

## FIRST: Identify Your Mode

The server exposes tools in one of three modes; **what you see in the tool list tells you which mode you're in**:

- **Code Mode** — you see exactly **three** tools: `netbox_search_schema`, `netbox_execute`, `rediscover_netbox`. Higher tiers default here. This is the primary, most efficient surface.
- **Discrete Mode** — you see ~34 `netbox_*` tools (plus plugin/product tools). Each operation is its own tool.
- **Both** — you see all of the above.

If you see `netbox_execute`, use Code Mode. If you only see discrete `netbox_*` tools, jump to [Discrete Mode](#discrete-mode).

---

## Code Mode

Instead of dozens of discrete tools (one round-trip each, bloated context), Code Mode exposes three tools whose arguments are **Python code**. You write a short program that calls injected NetBox helpers, does its own filtering/joining/looping in-process, and returns a single `result`. This collapses many tool round-trips into one.

### The three tools

| Tool | Argument | What it does |
|------|----------|--------------|
| `netbox_search_schema(code)` | Python reading a pre-loaded `schema` var, assigning `result` | **Discovery.** Local filtering of an in-memory schema dict — no NetBox I/O. Confirm object types, endpoints, filters before acting. |
| `netbox_execute(code)` | Python calling injected helpers, assigning `result` | **The workhorse.** Runs your program against NetBox and returns `result`. Its description is generated live, listing the exact helpers available in *this* deployment. |
| `rediscover_netbox()` | none | **Recovery only.** Re-probes NetBox and reconciles the tool inventory/schema. Use sparingly — see below. |

### The golden loop (minimize round trips)

1. **Search the schema first — don't guess.** Call `netbox_search_schema` to confirm the right object type, filter field, or special endpoint. Guessing object types/paths is the #1 failure mode.
2. **Write ONE `netbox_execute` program that does the whole job** — fetch, filter, join, loop, compute in a single program. Never call `netbox_execute` once per object when a loop or one list call would do.
3. **Always assign `result`.** If you never assign it you get back "Code ran successfully but did not assign to `result`" — a wasted turn.
4. **Project fields** with `fields=[...]` on reads to keep payloads (and the result-size cap) small.
5. **Prefer named helpers over raw `api()`** — they carry validation and clearer errors.

```python
# netbox_search_schema — confirm the type/filters first
result = {k: v for k, v in schema['object_types'].items() if v['app'] == 'dcim'}
```
```python
# netbox_execute — do the whole task in one program
sites = get('dcim.site', filters={'name': 'NYC-DC1'}, fields=['id', 'name'])
site_id = sites['results'][0]['id']
devs = get('dcim.device', filters={'site_id': site_id, 'status': 'active'},
           fields=['id', 'name', 'role', 'primary_ip4'], limit=1000)
result = {'site': 'NYC-DC1', 'count': devs['count'], 'devices': devs['results']}
```

### The sandbox contract (what your code may do)

`netbox_execute` / `netbox_search_schema` run through a validated sandbox. Respect these or your code is rejected:

- **NO `import` / `from ... import`.** This is the single most common failure — agents reach for `json`, `re`, `ipaddress`, `math`, `collections`. **Everything you need is already in the namespace or in allowed builtins.** Never write `import`.
- **NO class definitions, `del`, `async`/`await`/`yield`.**
- **NO dunder / introspection** (`eval`, `exec`, `open`, `getattr`, `globals`, `__class__`, `__subclasses__`, etc. are blocked).
- **Allowed builtins only** — the common ones: `len`, `str`, `int`, `float`, `bool`, `list`, `dict`, `set`, `tuple`, `range`, `enumerate`, `zip`, `sorted`, `sum`, `min`, `max`, `any`, `all`, `abs`, `round`, `map`, `filter`.
- **Loops, wall-clock time, and result size are capped** — keep programs bounded; project fields to stay under the result cap.
- **Must assign `result`.**
- **HTTP errors surface the NetBox response body**, including per-field validation errors — so a failed `create()`/`update()` returns actionable detail you can correct from and retry in the next program.

### Injected helpers

The exact set is rendered live in the `netbox_execute` description and gated by capability (write/IPAM-allocation/GraphQL/scripts/plugins/products). Object types use dot notation (`'dcim.device'`); the path form (`'dcim/devices'`) also resolves. Full catalog in [references/code-mode-helpers.md](references/code-mode-helpers.md) — load it when you need a specific signature. The essentials:

```
get(type, filters={}, fields=[], limit=50, offset=0, ordering=None) → {count, next, previous, results[]}
get_by_id(type, id, fields=[]) → object
search(query, object_types=[], limit=5) → {type: [results]}
options(type) → field schema (required fields, choices)
discover_models(app=None) / inspect_model(app, endpoint)        # enumerate types incl. plugins
create(type, data) / update(type, id, data) / delete(type, id)  # WRITE-gated
bulk_create / bulk_update / bulk_delete                          # WRITE-gated
available_prefixes / available_ips / allocate_prefix / allocate_ip ...   # IPAM-gated
graphql(query, variables=None, analyze=True)                     # GRAPHQL-gated
api(method, endpoint, params=None, data=None)                    # raw escape hatch
```

### Filter rules (two footguns)

1. **Never use `__in`.** NetBox silently ignores `field__in` on many fields and returns **all** rows — confidently wrong. The helpers reject it. Pass a **list as the field value** instead:
   ```python
   # WRONG
   get('dcim.site', filters={'id__in': [1, 2, 3]})
   # RIGHT — list value sends repeated query params, which NetBox honors
   get('dcim.site', filters={'id': [1, 2, 3]})
   ```
2. **No multi-hop traversal.** `device__site_id`-style nested filters are rejected. Use a **two-step query** — fetch the parent's ids, then filter the child:
   ```python
   racks = get('dcim.rack', filters={'site': 'nyc-dc1'}, fields=['id'])
   rack_ids = [r['id'] for r in racks['results']]
   devices = get('dcim.device', filters={'rack_id': rack_ids}, fields=['id', 'name'])
   result = devices['results']
   ```

### When to use `rediscover_netbox()`

Only when you have a **specific reason** to believe the tool inventory is stale: the user just installed/uninstalled a plugin, a tool you genuinely expected returns "tool not found," or a tool you no longer expect is still advertised. It is also the recovery path after a failed plugin registration. **Do NOT** call it before every operation, to "refresh" NetBox *data* (helpers are always live), or "just to be sure" at session start — each call costs a round-trip plus re-registration.

More end-to-end programs in [references/code-mode-workflows.md](references/code-mode-workflows.md).

---

## Discrete Mode

When Code Mode isn't available, every operation is its own MCP tool with a typed signature and a rich docstring. Discrete mode delivers more context directly (the tool list and docstrings are in front of you), so there is less to discover — but each operation is a separate round-trip. Core tools:

| Group | Tools |
|-------|-------|
| Reads | `netbox_get_objects`, `netbox_get_object_by_id`, `netbox_search_objects`, `netbox_get_changelogs` |
| Writes | `netbox_create_object`, `netbox_update_object`, `netbox_delete_object`, `netbox_bulk_create_objects`, `netbox_bulk_update_objects`, `netbox_bulk_delete_objects` |
| Connectivity | `netbox_trace_cable`, `netbox_get_cable_paths`, `netbox_get_rack_elevation`, `netbox_get_connected_device` |
| IPAM | `netbox_get_available_*` (prefixes/ips/vlans/asns), `netbox_allocate_next_*` |
| Automation/introspection | `netbox_render_config`, `netbox_run_script`, `netbox_graphql_query`, `netbox_get_object_schema`, `discover_models`, `inspect_model`, `rediscover_netbox` |

Discrete-mode rules the docstrings already enforce (follow them):
- **Always pass `fields=`** to minimize tokens (80–90% payload reduction on reads). `limit` default 50, max 1000.
- The **`__in` and multi-hop rules are identical** to code mode — pass a list value; use the two-step pattern.
- `netbox_get_changelogs`: `action` must be **imperative singular** (`"create"`/`"update"`/`"delete"` — NetBox rejects `"updated"` with 400). `changed_object_type` is `"app.model"`; `changed_object_type_id` is the integer ContentType PK.
- Object types accept both `"dcim.device"` and `"dcim/devices"`. Use `discover_models()` to enumerate types (including plugins).

## Product & Plugin Namespaces

When the tenant is entitled and the integration is configured, additional helpers/tools appear:
- **Plugins:** branching (`branch_*`), changes (`change_request_*`, reviews, comments), custom-objects (`custom_type_*`/`custom_object_*`).
- **Products:** **Assurance** (`assurance_*`) and **Discovery** (`discovery_*`).

These are **entitlement-gated** — present only when your tenant has the product/plugin. Don't assume they exist; discover them. For their domain logic, see [netbox-branching](../netbox-branching/SKILL.md), [netbox-changes](../netbox-changes/SKILL.md), [netbox-custom-objects](../netbox-custom-objects/SKILL.md), [netbox-assurance](../netbox-assurance/SKILL.md), and [netbox-discovery](../netbox-discovery/SKILL.md).

## Anti-Patterns

1. **Writing `import`** in code mode → "Forbidden operation: Import." The dominant failure. Use namespace helpers + allowed builtins only.
2. **Not searching the schema first** → guessing a wrong object type or unsupported filter. Confirm via `netbox_search_schema` before executing.
3. **The `__in` footgun** → silently-broad results / rejection. Use a list as the field value.
4. **Multi-hop filters** → rejected. Use the two-step pattern.
5. **Forgetting to assign `result`** → wasted turn.
6. **One `netbox_execute` per object** instead of a single looped/bulk program → defeats the point of code mode.
7. **Over-calling `rediscover_netbox`** as a precaution → it's a round-trip cost, not a data refresh.
8. **Not projecting `fields=`** → oversized payloads, hitting the result cap, wasted tokens (both modes).
9. **Assuming a product/plugin namespace exists** → it's entitlement-gated. Discover it.
10. **`NameError` on an undefined name** (`ip_network`, `re`, …) → only injected helpers + allowed builtins exist.

## References

- [references/code-mode-helpers.md](references/code-mode-helpers.md) — Full injected-helper catalog with signatures, grouped by capability gate. Load when you need an exact signature.
- [references/code-mode-workflows.md](references/code-mode-workflows.md) — End-to-end one-program examples (reads, list-of-ids, two-step joins, IPAM allocation, changelog audits).
