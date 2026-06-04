# Context Variables Reference

Complete reference for variables available during NetBox config template rendering.

## Device / VM Rendering (`render-config`)

Endpoint: `POST /api/dcim/devices/{id}/render-config/`

The rendering context is built in three layers, merged in order:

### Layer 1: Config Context Data

All matching config contexts are merged (by weight, then name) into a flat dict. These appear as **top-level template variables**:

```jinja2
{# If config context contains {"ntp_servers": ["10.0.0.1"], "domain": "example.com"} #}
{{ domain }}              → "example.com"
{{ ntp_servers[0] }}      → "10.0.0.1"
```

### Layer 2: POST Body Data

Any JSON keys sent in the POST request body are merged on top of config context data. This **overrides** config context values on key collision.

```python
# POST body: {"domain": "override.com"}
# Template receives domain = "override.com" (not the config context value)
```

### Layer 3: Object Instance

The device or VM instance is added as a single key:

| Rendering Target | Variable Name | Type |
|-----------------|---------------|------|
| Device | `device` | `dcim.Device` instance |
| Virtual Machine | `virtualmachine` | `virtualization.VirtualMachine` instance |

### Accessing Device Relations

The `device` variable is a full ORM instance. All relations are traversable:

```jinja2
{# Identity #}
{{ device.name }}                    → "switch01"
{{ device.serial }}                  → "FDO12345678"
{{ device.asset_tag }}               → "ASSET-001"

{# Classification #}
{{ device.role.name }}               → "Access Switch"
{{ device.role.slug }}               → "access-switch"
{{ device.device_type.model }}       → "C9300-48T"
{{ device.device_type.manufacturer.name }} → "Cisco"
{{ device.platform.name }}           → "IOS-XE"
{{ device.platform.slug }}           → "ios-xe"

{# Location #}
{{ device.site.name }}               → "NYC-DC1"
{{ device.site.slug }}               → "nyc-dc1"
{{ device.location.name }}           → "Row A"
{{ device.rack.name }}               → "Rack 42"

{# Tenancy #}
{{ device.tenant.name }}             → "Acme Corp"

{# IP Addressing #}
{{ device.primary_ip4.address }}     → "10.0.1.1/24"
{{ device.primary_ip6.address }}     → "2001:db8::1/64"
{{ device.oob_ip.address }}          → "192.168.1.1/24"

{# Related Objects (querysets) #}
{% for iface in device.interfaces.all() %}
{% for ip in iface.ip_addresses.all() %}
{% for vc in device.virtual_chassis.members.all() %}
```

### Accessing Virtual Machine Relations

```jinja2
{{ virtualmachine.name }}
{{ virtualmachine.cluster.name }}
{{ virtualmachine.cluster.type.name }}
{{ virtualmachine.role.name }}
{{ virtualmachine.platform.slug }}
{{ virtualmachine.site.name }}
{{ virtualmachine.tenant.name }}
{{ virtualmachine.primary_ip4.address }}
{% for iface in virtualmachine.interfaces.all() %}
```

## General-Purpose Rendering (`config-templates/{id}/render/`)

Endpoint: `POST /api/extras/config-templates/{id}/render/`

No device or VM is in context. Instead, the template receives:

### All Public NetBox Model Classes

Organized by app label, enabling direct ORM queries:

```jinja2
{# Query any NetBox model #}
{% set sites = dcim.Site.objects.all() %}
{% set devices = dcim.Device.objects.filter(site__slug='nyc-dc1') %}
{% set prefixes = ipam.Prefix.objects.filter(role__slug='management') %}
{% set vms = virtualization.VirtualMachine.objects.filter(status='active') %}
{% set tenants = tenancy.Tenant.objects.all() %}

{# Aggregations #}
{{ dcim.Device.objects.count() }}
{{ ipam.IPAddress.objects.filter(status='active').count() }}
```

Available app labels include: `dcim`, `ipam`, `virtualization`, `tenancy`, `circuits`, `extras`, `wireless`, `vpn`.

### POST Body Data

Any JSON sent in the request body appears as top-level variables:

```python
requests.post(
    f"{NETBOX}/api/extras/config-templates/5/render/",
    headers=headers,
    json={"region": "us-east", "vlans": [10, 20, 30]},
)
```

```jinja2
Region: {{ region }}
{% for vlan in vlans %}
vlan {{ vlan }}
{% endfor %}
```

## Jinja2 Environment Details

### Sandbox Restrictions

Templates run in `jinja2.sandbox.SandboxedEnvironment`. You **cannot**:
- Import Python modules
- Access file system
- Execute system commands
- Access dunder attributes on unsafe objects

### Available Filters

The standard Jinja2 built-in filters are available:
`abs`, `attr`, `batch`, `capitalize`, `center`, `default`, `dictsort`, `escape`, `filesizeformat`, `first`, `float`, `forceescape`, `format`, `groupby`, `indent`, `int`, `items`, `join`, `last`, `length`, `list`, `lower`, `map`, `max`, `min`, `pprint`, `random`, `reject`, `rejectattr`, `replace`, `reverse`, `round`, `safe`, `select`, `selectattr`, `slice`, `sort`, `string`, `striptags`, `sum`, `title`, `trim`, `truncate`, `unique`, `upper`, `urlencode`, `urlize`, `wordcount`, `wordwrap`, `xmlattr`.

**NetBox 4.6.2+** adds a built-in **`env()`** filter that returns a system environment variable's value: `{{ 'WEBHOOK_TOKEN_3' | env }}`. It only resolves names matched by the `JINJA_ENVIRONMENT_PARAMS` config allowlist (fnmatch wildcards); any other name returns `None`. On 4.6.1 and earlier, only the standard filters above are built in.

Custom filters can be registered in NetBox configuration:

```python
# configuration.py
JINJA2_FILTERS = {
    "ipaddr": "netaddr.IPAddress",  # example custom filter
}
```

### Environment Parameters

Each template can customize Jinja2 behavior via `environment_params` (JSON field):

```json
{
    "undefined": "jinja2.StrictUndefined",
    "trim_blocks": true,
    "lstrip_blocks": true
}
```

`StrictUndefined` is recommended for production templates — raises errors on undefined variables instead of silently rendering empty strings.

> **NetBox 4.6.1+ — allowlist (CVE-2026-29514).** `environment_params` keys are restricted to a fixed allowlist (`JINJA_ENV_PARAMS_ALLOWED`) after an RCE fix shared by ConfigTemplate and ExportTemplate. The delimiter/whitespace/scalar params (incl. `trim_blocks`, `lstrip_blocks`, `autoescape`, `keep_trailing_newline`) are allowed, and `undefined` must be one of `jinja2.StrictUndefined`, `jinja2.Undefined`, `jinja2.ChainableUndefined`, `jinja2.DebugUndefined`. **`extensions`, `finalize`, `loader`, and `bytecode_cache` are blocked** — a template setting them is rejected. The example above remains valid.
