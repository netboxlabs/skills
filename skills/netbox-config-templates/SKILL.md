---
name: netbox-config-templates
description: >
  How to use NetBox's configuration template system for generating device configs.
  Covers Jinja2 templates, config contexts, template rendering API, and common patterns.
  Use when building config generation workflows, automating device provisioning, or
  integrating NetBox with config management tools.
license: Apache-2.0
---

# NetBox Config Templates

> **Your knowledge of NetBox config templates may be outdated.** Template context variables, rendering API, and config context merge behavior evolve between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Config templates docs | `https://netboxlabs.com/docs/netbox/customization/config-templates/` | Template syntax, context variables |
| Config contexts docs | `https://netboxlabs.com/docs/netbox/customization/config-contexts/` | Hierarchical data, merge rules |
| NetBox repo | `https://github.com/netbox-community/netbox` | Template rendering source code |
| NetBox MCP server | If configured — fetch device config contexts to test template rendering | Verify context data |

NetBox provides a built-in Jinja2-based configuration rendering system. You define templates, populate data through config contexts, and render per-device or per-VM configurations via the API. This skill covers the entire workflow.

## When to Use This Skill

- Generating device/VM configurations from NetBox data
- Setting up config contexts (hierarchical data) for template rendering
- Integrating NetBox config rendering into CI/CD or provisioning pipelines
- Troubleshooting template rendering issues or config context merge behavior

## Quick Reference

### Template Resolution Order (Device)

```
device.config_template → device.role.config_template → device.platform.config_template
```

First match wins. Only one template renders per device. VMs follow the same pattern (role → platform).

### Key API Endpoints

| Action | Method | Endpoint |
|--------|--------|----------|
| Render device config | POST | `/api/dcim/devices/{id}/render-config/` |
| Render VM config | POST | `/api/virtualization/virtual-machines/{id}/render-config/` |
| Render template directly | POST | `/api/extras/config-templates/{id}/render/` |
| CRUD config templates | GET/POST/PUT/PATCH/DELETE | `/api/extras/config-templates/` |
| CRUD config contexts | GET/POST/PUT/PATCH/DELETE | `/api/extras/config-contexts/` |

### Context Variables Cheat Sheet

**In device/VM rendering** (`render-config`):

| Variable | Type | Description |
|----------|------|-------------|
| `device` / `virtualmachine` | ORM instance | Full model with all relations |
| *(config context keys)* | varies | Merged config context data as top-level keys |
| *(POST body keys)* | varies | Extra data sent with the render request |

**In general-purpose rendering** (`config-templates/{id}/render/`):

| Variable | Type | Description |
|----------|------|-------------|
| `dcim`, `ipam`, `virtualization`, etc. | module-like | Access to all public NetBox model classes |
| *(POST body keys)* | varies | Extra data sent with the render request |

See [references/context-variables.md](references/context-variables.md) for the complete breakdown.

## Config Context System

Config contexts provide structured JSON data to templates. They are assigned based on object attributes and merged by weight.

### Assignment Logic

A config context applies when the object matches **all** non-empty assignment categories (AND across categories) and **any** value within each category (OR within a category).

Assignment categories: Regions, Site groups, Sites, Locations, Device types, Roles, Platforms, Cluster types, Cluster groups, Clusters, Tenant groups, Tenants, Tags.

### Merge Order

1. All matching **active** config contexts, ordered by **weight ascending**, then **name**
2. Higher-weight contexts overwrite lower-weight on key collision
3. **Local context data** (on the device/VM itself) merges **last** and always wins

### ⚠️ Lists Are Replaced, Not Merged

`deepmerge()` merges dicts recursively but **replaces** non-dict values entirely:

```json
// Weight 100: {"ntp_servers": ["10.0.0.1", "10.0.0.2"]}
// Weight 200: {"ntp_servers": ["10.1.1.1"]}
// Result:     {"ntp_servers": ["10.1.1.1"]}  ← NOT concatenated
```

Design data structures with this in mind. Use dicts with unique keys when you need additive merging.

### Hierarchy Resolution

For hierarchical models (Region, Site Group, etc.), a context assigned to a parent also applies to all descendants. A context assigned to "Americas" region matches devices in "US-East" sub-region.

For a deep dive on merging, see [references/config-context-merging.md](references/config-context-merging.md). For config context data modeling best practices, see [netbox-data-modeling](../netbox-data-modeling/SKILL.md).

## Writing Templates

### Accessing Device Data

The `device` (or `virtualmachine`) variable is a full Django ORM instance:

```jinja2
hostname {{ device.name }}
!
{% if device.primary_ip4 %}
ip address {{ device.primary_ip4.address | ipaddr('address') }}
{% endif %}
!
{% for iface in device.interfaces.all() %}
interface {{ iface.name }}
  description {{ iface.description | default('', true) }}
  {% for ip in iface.ip_addresses.all() %}
  ip address {{ ip.address }}
  {% endfor %}
{% endfor %}
```

### Using Config Context Data

Config context keys appear as top-level template variables:

```jinja2
{% for server in ntp_servers | default([]) %}
ntp server {{ server }}
{% endfor %}
!
{% for server in syslog_servers | default([]) %}
logging host {{ server }}
{% endfor %}
!
{% if snmp_community is defined %}
snmp-server community {{ snmp_community }} RO
{% endif %}
```

### Template Inheritance (DataSource Required)

`{% extends %}` and `{% include %}` only work when templates are stored in a **DataSource** (git repo, S3, etc.). Templates defined with inline `template_code` cannot resolve other templates.

```jinja2
{# Stored in a DataSource alongside base.j2 #}
{% extends 'base.j2' %}

{% block interfaces %}
{% for iface in device.interfaces.all() %}
interface {{ iface.name }}
{% endfor %}
{% endblock %}
```

### ORM Queries in Templates

Both device and general-purpose rendering expose NetBox models:

```jinja2
{# Query prefixes from IPAM #}
{% set mgmt_prefixes = ipam.Prefix.objects.filter(role__slug='management') %}
{% for pfx in mgmt_prefixes %}
ip route {{ pfx.prefix }} Management
{% endfor %}
```

**Warning:** ORM queries run unsandboxed against the database. A careless `.all()` on a large table will be slow. There is no query limiting.

### Jinja2 Environment

- Templates run in a **SandboxedEnvironment** — no file I/O, no arbitrary imports
- Custom filters can be added via `JINJA2_FILTERS` in NetBox configuration
- **Built-in `env()` filter (NetBox 4.6.2+)** — returns a system environment variable's value: `{{ 'WEBHOOK_TOKEN_3' | env }}`. Gated by the `JINJA_ENVIRONMENT_PARAMS` config list (an fnmatch wildcard allowlist of permitted variable names); returns `None` for any name not matched. On 4.6.1 and earlier there are no built-in custom filters — only standard Jinja2 filters.
- Per-template `environment_params` can customize behavior (e.g., `undefined: jinja2.StrictUndefined`), but on **NetBox 4.6.1+** the keys are **allowlisted** — see below.

> **NetBox 4.6.1+ — `environment_params` allowlist (CVE-2026-29514).** A patched RCE via `environment_params` (shared by ConfigTemplate and ExportTemplate) restricts which keys are accepted (`JINJA_ENV_PARAMS_ALLOWED`). Allowed: the standard delimiter/whitespace/scalar params (`trim_blocks`, `lstrip_blocks`, `autoescape`, `keep_trailing_newline`, block/comment/variable `*_string` delimiters, etc.) and `undefined` — but `undefined` must be one of `jinja2.StrictUndefined`, `jinja2.Undefined`, `jinja2.ChainableUndefined`, `jinja2.DebugUndefined`. **`extensions`, `finalize`, `loader`, and `bytecode_cache` are blocked** and a template setting them is rejected. The skill's examples (`StrictUndefined`, `trim_blocks`, `lstrip_blocks`) remain valid; do not recommend the blocked keys.

## Common Vendor Patterns

### Strategy 1: Platform-Based Template Assignment (Recommended)

Assign different config templates to different Platforms:

| Platform | Config Template |
|----------|----------------|
| `junos` | `junos-base.j2` |
| `ios-xe` | `ios-xe-base.j2` |
| `eos` | `eos-base.j2` |

Template resolves via: device → role → **platform**. This is the cleanest approach.

### Strategy 2: Conditional Logic in a Single Template

```jinja2
{% if device.platform.slug == 'junos' %}
system {
    host-name {{ device.name }};
}
{% elif device.platform.slug == 'ios-xe' %}
hostname {{ device.name }}
{% elif device.platform.slug == 'eos' %}
hostname {{ device.name }}
{% endif %}
```

Works for small differences but becomes unwieldy. Prefer Strategy 1 for production.

## API Rendering Patterns

### Render a Device Config

```python
import requests

NETBOX = "https://netbox.example.com"
TOKEN = "nbt_abc123.xxxxxxxxxxxxxxxx"
headers = {"Authorization": f"Bearer {TOKEN}"}

# Render config for device ID 42
response = requests.post(
    f"{NETBOX}/api/dcim/devices/42/render-config/",
    headers=headers,
    json={},  # optional extra context data
)
response.raise_for_status()
config = response.json()["content"]
```

Use `Accept: text/plain` header to get raw config output without JSON wrapping.

### Override Config Context at Render Time

```python
# POST body merges after config context (overrides on collision)
response = requests.post(
    f"{NETBOX}/api/dcim/devices/42/render-config/",
    headers=headers,
    json={"ntp_servers": ["10.99.99.1"]},  # overrides config context
)
```

### Render a Template Directly (No Device)

```python
# General-purpose rendering — no device/VM in context
response = requests.post(
    f"{NETBOX}/api/extras/config-templates/5/render/",
    headers=headers,
    json={"hostname": "switch01", "vlans": [10, 20, 30]},
)
```

For detailed API examples including pynetbox and error handling, see [references/api-rendering-patterns.md](references/api-rendering-patterns.md).

## Pitfalls

### Performance

- **Exclude config_context from list endpoints:** `GET /api/dcim/devices/?exclude=config_context` — computing merged config context per device is expensive. Always exclude on bulk queries.
- **ORM queries in templates** have no guardrails — avoid unbounded `.all()` on large tables.

### Template Behavior

- **Template resolution is first-match only** — device → role → platform. A device still has one *assigned* template. **NetBox 4.6+** lets you render any other template against a device's context without changing assignments by passing `config_template_id` in the render-config request body (or `?config_template_id=<id>` on the UI render URL) — useful for CI/dry-run rendering of alternative templates. (Requires `view` permission on Config Template plus `render_config` on the device.)
- **No template versioning** — templates are mutable. Only NetBox's change log provides history.
- **Render debugging (NetBox 4.6+)** — when developing templates in a script/plugin, call `render_jinja2(..., debug=True)` to enable Jinja2's `debug` extension (`{% debug %}`) for inspecting the available context.
- **extends/include requires DataSource** — inline template_code cannot reference other templates.

### Config Context Surprises

- **Lists replace, don't append** — the most common surprise. See merge rules above.
- **POST data silently overrides config context** — intentional but can be confusing when debugging unexpected values.
- **environment_params vary per template** — undefined variable handling may differ between templates if configured differently.

## References

- [references/context-variables.md](references/context-variables.md) — Complete context variable reference for all rendering modes
- [references/config-context-merging.md](references/config-context-merging.md) — Deep dive on hierarchy, weight, and merge behavior
- [references/api-rendering-patterns.md](references/api-rendering-patterns.md) — Detailed API examples with pynetbox and requests
