---
name: netbox-custom-objects
description: >
  No-code data model extensibility for NetBox. Create custom object types with typed fields,
  relationships to core models and other custom objects, full REST API, and standard NetBox
  features (tags, change logging, search, bookmarks, journaling).
license: Apache-2.0
---

# NetBox Custom Objects

> **Your knowledge of Custom Objects may be outdated.** Field types, relationship options, branching behavior, and API surface change between plugin releases (0.5 → 0.7 added polymorphic fields, portable schema, branching, GraphQL, config context, `coordinates`, `display_expression`). Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Custom Objects docs | `https://netboxlabs.com/docs/custom-objects/` | Overview, workflow, config context, config templates |
| Field attributes | `https://netboxlabs.com/docs/custom-objects/field-attributes/` | Per-type attributes |
| REST API | `https://netboxlabs.com/docs/custom-objects/rest-api/` | Endpoints, payloads |
| GraphQL | `https://netboxlabs.com/docs/custom-objects/graphql/` | Query field naming, type mapping |
| Branching | `https://netboxlabs.com/docs/custom-objects/branching/` | Version gate, branch behavior |
| Portable schema | `https://netboxlabs.com/docs/custom-objects/portable-schema/` | Export/preview/apply |
| Release notes | `https://github.com/netboxlabs/netbox-custom-objects/blob/main/docs/releases.md` | What changed in the installed version |
| NetBox MCP server | If configured — list existing custom object types and instances | Schema discovery |
| NetBox Platform MCP | If configured — full CRUD on custom objects | Create types, manage instances |

## FIRST: Verify Connectivity and Version

```bash
# Plugin installed? (404 = not installed; 403 = token lacks custom_objects permissions)
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/plugins/custom-objects/custom-object-types/" | python -m json.tool

# Which plugin version? (core /api/status/ lists installed plugins and versions)
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/status/" | python -c "import sys,json; d=json.load(sys.stdin); print(d['netbox-version'], d['plugins'].get('netbox_custom_objects'))"
```

> **Version note:** this skill targets plugin **v0.7.x** (latest v0.7.0, 2026-09-15), which supports **NetBox 4.5.2 through 4.7.x**. Behaviors that differ by plugin version are tagged inline (`v0.5+`, `v0.6+`, `v0.6.1+`, `v0.7+`); features that need NetBox 4.7 are tagged **NetBox 4.7+**. Always check the installed version first — the branching model changed completely in 0.6.0.

---

Extend the NetBox data model without writing code. Custom Objects let administrators define new object types with typed fields, validation rules, and relationships — through the UI, REST API, GraphQL (read), or a portable schema document.

## When to Use This Skill

- Creating new object types in NetBox (e.g., DHCP Scopes, Contracts, Applications)
- Adding fields to custom object types (text, numeric, object references, coordinates, etc.)
- Building integrations that create or consume custom objects via REST, GraphQL, or pynetbox
- Automating custom object type provisioning (portable schema export/preview/apply) in CI/CD
- Querying linked objects (reverse lookups from core objects to custom objects)
- Using custom objects in branches, config contexts, or device config templates

## Quick Reference

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Custom Object Type (COT)** | A new model definition — like a database table. Has `name` (internal), `slug` (URL/API path), display names, fields. |
| **Custom Object Type Field (COTF)** | A column on a COT — typed, with validation, ordering, grouping. |
| **Custom Object** | An instance of a COT — like a row in the table. |
| **Primary field** | The COTF whose value becomes the object's display name (default name is `<Type> <id>`). |
| **Display expression** *(v0.6.1+)* | Jinja2 template on the COT (`display_expression`) that composes the display name from several fields; overrides the primary field when set. |
| **Context field** | A COTF with `context: true` — its value is shown alongside the object wherever another object references it (and in the REST `_context` key). |
| **Group name** | Groups related COTs in the navigation sidebar (on COT) or fields in forms (on COTF). |

### API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/api/plugins/custom-objects/custom-object-types/` | CRUD on type definitions (`?slug=` filter works, v0.5.2+) |
| `/api/plugins/custom-objects/custom-object-type-fields/` | CRUD on field definitions |
| `/api/plugins/custom-objects/{slug}/` | CRUD on instances (one endpoint per type, keyed by **slug**) |
| `/api/plugins/custom-objects/linked-objects/` | Reverse lookups: which custom objects reference a given object |
| `/api/plugins/custom-objects/schema/preview/` | Diff a portable schema document against the live DB (read-only) |
| `/api/plugins/custom-objects/schema/apply/` | Apply a portable schema document (atomic; needs add+change on COT) |
| `/graphql/` *(v0.6+)* | Read custom objects as `custom_objects_<slug>` / `custom_objects_<slug>_list` |

### Field Types (14)

| Type | Description | Type-specific options |
|------|-------------|-------------------|
| `text` | Short text | `validation_regex` |
| `longtext` | Multi-line text (textarea) | `validation_regex` |
| `integer` | Whole number, **64-bit** (v0.6+) | `validation_minimum`, `validation_maximum` |
| `decimal` | Decimal number | `validation_minimum`, `validation_maximum` |
| `boolean` | True/false | — (no `unique`) |
| `date` | ISO 8601 date (`YYYY-MM-DD`) | — |
| `datetime` | ISO 8601 datetime | — |
| `url` | URL plus optional link title *(v0.7+: REST exposes `<name>` and `<name>_title`)* | `validation_regex` |
| `json` | Arbitrary JSON | — |
| `select` | Single choice from a choice set | `choice_set` (required) |
| `multiselect` | Multiple choices from a choice set | `choice_set` (required) |
| `object` | Reference to one object | `app_label`+`model` or `is_polymorphic`+`related_object_types_input`; `related_object_filter`, `related_name`, `on_delete_behavior` |
| `multiobject` | Reference to many objects | same as `object` minus `on_delete_behavior`; no `unique` |
| `coordinates` *(v0.6+)* | Latitude/longitude pair; REST exposes `<name>_latitude` and `<name>_longitude` (both-or-neither) | — (no `unique`, no `default`) |

> The release notes call the coordinates type "location"; the API `type` value is **`coordinates`**.

### What changed since v0.5.1

| Version | Change |
|---------|--------|
| 0.5.2 | `related_name` works on polymorphic fields (reverse accessor); `?slug=` filter on COT list; tags on POST/PATCH now persist; select fields display labels not raw keys; `null` accepted for optional object fields |
| 0.6.0 | **Branching**: COTs, fields, and instances are branch-aware (gated: NetBox ≥ 4.6.2, netbox-branching ≥ 1.0.4); **GraphQL** read support; config context support (`config_context_enabled`); Quick Add on object/multiobject form fields; contacts; ownership (`owner`); 64-bit integers; `coordinates` type |
| 0.6.1 | `display_expression`; **NetBox 4.7+** config-template access (`custom_objects.<name>`, `'<name>' \| custom_objects`); bulk-edit "Set null"; `?id=` and lookup suffixes (`__gt`, `__ic`, …) fixed on instance filters; NetBox 4.7 support |
| 0.7.0 | "Custom Objects" related tab on every referenced object (permission-filtered); URL link title; schema preview/apply accept and return YAML; context fields shown in global search |

## Creating a Custom Object Type

### Step 1: Create the Type

```http
POST /api/plugins/custom-objects/custom-object-types/
```

```json
{
  "name": "dhcp_scope",
  "slug": "dhcp-scopes",
  "description": "DHCP address scopes",
  "verbose_name": "DHCP Scope",
  "verbose_name_plural": "DHCP Scopes",
  "group_name": "IPAM Extensions",
  "version": "1.0.0",
  "display_expression": "{{ scope_name }}{% if vlan %} (VLAN {{ vlan }}){% endif %}",
  "config_context_enabled": false
}
```

**Type properties:**

| Property | Notes |
|----------|-------|
| `name` | Internal name: `^[a-z0-9]+(_[a-z0-9]+)*$` — lowercase, underscores, no leading/trailing/double underscores. Unique. Used by config templates. |
| `slug` | URL-safe identifier. Used in the REST path, GraphQL field name, `CUSTOM_VALIDATORS` key, and cross-COT references. Unique. |
| `verbose_name` / `verbose_name_plural` | Display names. Default to title-cased `name` / `verbose_name + "s"`. |
| `group_name` | Optional. Groups this type with others in the sidebar. |
| `version` | Optional PEP 440 string; used by the portable schema feature. |
| `display_expression` *(v0.6.1+)* | Optional Jinja2 template rendered with field values by name. Undefined fields render as empty strings; a render error silently falls back to the primary field. The UI form validates Jinja syntax; the REST API does not. |
| `config_context_enabled` *(v0.6+)* | **Create-only.** Adds `local_context_data` to every instance and a Config Context tab. Cannot be toggled later (API returns a validation error). |

### Step 2: Add Fields

```http
POST /api/plugins/custom-objects/custom-object-type-fields/
```

```json
{
  "custom_object_type": 1,
  "name": "scope_name",
  "label": "Scope Name",
  "type": "text",
  "required": true,
  "unique": true,
  "primary": true,
  "validation_regex": "^[a-zA-Z0-9_-]+$"
}
```

**Key field properties:**

| Property | Purpose | Notes |
|----------|---------|-------|
| `primary` | Use this field's value as the object's display name | One per type; `display_expression` on the COT overrides it |
| `context` | Show this value as extra context wherever the object is referenced | Appears in REST as `_context`; surfaced in global search (v0.7+) |
| `required` | Field must have a value | Enforced on create and update (API and UI) |
| `unique` | Value must be unique across all instances | Not allowed on `boolean`, `multiobject`, `coordinates` |
| `weight` | Display ordering (lower = higher) | Default: 100 |
| `group_name` | Visual grouping in forms | Fields with same group_name appear together |
| `search_weight` | Search ranking (100=high, 500=medium, 1000=low, 0=excluded) | Default: 500 |
| `filter_logic` | `loose` (substring), `exact`, `disabled` | Default: `loose`; affects the REST filter for text-like fields |
| `ui_visible` / `ui_editable` | `always`/`if-set`/`hidden` and `yes`/`no`/`hidden` | Hidden fields are still writable via the API |
| `default` | Default value, stored as JSON | **REST:** send the JSON value directly (`"default": "active"`, `"default": 86400`, `"default": true`). **UI form:** type a JSON literal (strings need quotes). Not allowed on `coordinates`. |
| `is_cloneable` | Include when cloning objects | Default: false |
| `deprecated`, `deprecated_since`, `scheduled_removal` | Mark a field read-only during a migration grace period | PEP 440 strings; consumed by portable schema |
| `schema_id` | Read-only stable id used by portable schema | Auto-assigned |

### Step 3: Create Instances

```http
POST /api/plugins/custom-objects/dhcp-scopes/
```

```json
{
  "scope_name": "office-floor-2",
  "ip_range": 5,
  "vlan": 100,
  "tags": [{"name": "production"}],
  "owner": 3
}
```

The endpoint path uses the COT's **slug** (not name). Every instance also carries `id`, `url`, `display`, `owner` *(v0.6+, NetBox ownership — omit if unused)*, `tags`, `created`, `last_updated`, plus `_context` when the type has context fields and `local_context_data` when config context is enabled.

## Object Reference Fields

Reference core NetBox objects, plugin models, or other custom objects using `object` and `multiobject` fields.

### Referencing Core Objects

```json
{
  "custom_object_type": 1,
  "name": "site",
  "label": "Site",
  "type": "object",
  "app_label": "dcim",
  "model": "site",
  "related_name": "dhcp_scopes",
  "on_delete_behavior": "protect"
}
```

Common `app_label.model` combinations: `dcim.device`, `dcim.site`, `dcim.rack`, `dcim.interface`, `ipam.prefix`, `ipam.ipaddress`, `ipam.vlan`, `ipam.iprange`, `tenancy.tenant`, `circuits.circuit`, `virtualization.virtualmachine`.

- `on_delete_behavior` *(v0.5+)*: `set_null` (default), `cascade`, or `protect` — what happens to the custom object when the referenced object is deleted. **`object` fields only**; ignored on `multiobject`.
- `related_name`: reverse accessor on the target (e.g. `site.dhcp_scopes.all()` in export/config templates). Works on polymorphic fields too *(v0.5.2+)*. Must be unique per target type.
- The response echoes `related_object_type` (read-only nested `{id, app_label, model}`); `app_label`/`model` are write-only.

### Referencing Other Custom Objects

Set `app_label` to `"custom-objects"` and `model` to the **target COT's slug**:

```json
{
  "custom_object_type": 2,
  "name": "parent_scope",
  "label": "Parent Scope",
  "type": "object",
  "app_label": "custom-objects",
  "model": "dhcp-scopes"
}
```

> Self-referential fields (a COT pointing to itself) are allowed. Circular references between different COTs are detected and rejected with a 400 — including through polymorphic fields *(v0.5.2+)*.

### Polymorphic Reference Fields

An `object`/`multiobject` field can reference **multiple** object types *(v0.5+)*. Set `is_polymorphic: true` and list allowed types in `related_object_types_input` instead of `app_label`/`model`:

```json
{
  "custom_object_type": 9,
  "name": "linked_resource",
  "label": "Linked Resource",
  "type": "object",
  "is_polymorphic": true,
  "related_object_types_input": [
    {"app_label": "dcim", "model": "device"},
    {"app_label": "custom-objects", "model": "servers"}
  ]
}
```

Write an instance value as a dict: `{"app_label": "dcim", "model": "device", "object_id": 7}` (`content_type_id` and `id` are accepted aliases, so a read representation round-trips). For polymorphic `multiobject`, pass a list of such dicts. `is_polymorphic` and the allowed types are **immutable** after creation — delete and recreate the field to change them.

### Filtering Object Selections

Narrow the UI drop-down with `related_object_filter` (a `query_params` dict):

```json
{"type": "object", "app_label": "dcim", "model": "device", "related_object_filter": {"status": "active"}}
```

> *(v0.6+)* Object and multiobject form fields support NetBox **Quick Add**, so users can create the referenced object inline.

## Linked Objects (Reverse Lookups)

Find all custom objects that reference a given object (core or custom):

```http
GET /api/plugins/custom-objects/linked-objects/?object_type=dcim.device&object_id=42
```

```json
{
  "count": 1,
  "results": [
    {
      "custom_object_type": {"id": 1, "name": "server", "slug": "servers"},
      "field_name": "primary_device",
      "object": {"id": 7, "display": "web-server-01"}
    }
  ]
}
```

Both parameters are required. In the UI *(v0.7+)*, every referenced object gets a **Custom Objects** tab listing the same relationships across all types, with search, type/tag filters, and a count badge — filtered to the viewer's per-type view permissions (the tab hides when there is nothing they may see). `max_multiobject_display` (plugin config, default 3) caps how many multiobject targets show per row.

## Filtering Instances

```http
GET /api/plugins/custom-objects/dhcp-scopes/?scope_name=office        # text: substring (filter_logic=loose)
GET /api/plugins/custom-objects/dhcp-scopes/?lease_time__gte=3600       # numeric/date lookups (v0.6.1+)
GET /api/plugins/custom-objects/dhcp-scopes/?site=5&site_id=5          # object fields: either form
GET /api/plugins/custom-objects/dhcp-scopes/?target_dcim_device=7      # polymorphic: <field>_<app>_<model>
GET /api/plugins/custom-objects/dhcp-scopes/?id=1&id=2&tag=production&q=office
```

- Text-like fields (`text`, `longtext`, `url`, `json`) default to `icontains`; set `filter_logic: "exact"` to get exact matching plus NetBox's suffix lookups (`__ic`, `__isw`, `__n`, …). `filter_logic: "disabled"` removes the filter.
- Standard NetBox params (`q`, `tag`, `created__gte`, `limit`/`offset`, `ordering`) apply. See `/api/schema/swagger-ui/` for the generated filter list per type.

## GraphQL (v0.6+, read-only)

```graphql
query {
  custom_objects_dhcp_scopes_list { id display scope_name site { id name region { name } } }
  custom_objects_dhcp_scopes(id: 42) { id display }
}
```

- Root fields are `custom_objects_<slug>` and `custom_objects_<slug>_list`, with non-identifier characters in the slug replaced by `_`. Bare slug names (`dhcp_scopes_list`) do not exist.
- Relationship fields resolve to the target's native NetBox GraphQL type (traversable); polymorphic fields are unions — use `... on DeviceType { }` fragments.
- `coordinates` fields and URL titles are not exposed in GraphQL; use REST.
- Mutations and server-side field filters are not present in the 0.7.0 schema — write and filter via REST.
- Details: [references/graphql-and-branching.md](references/graphql-and-branching.md).

## Branching

The model **flipped in v0.6.0**. Check the installed plugin version before advising.

| Plugin | COT / field changes on a branch | Instance writes on a branch | Schema apply on a branch |
|--------|----------------------------------|-----------------------------|--------------------------|
| **v0.6+** | Branch-aware: tracked, merged, revertible (field adds/renames/deletes, type deletes) | **Allowed** and tracked like any NetBox object | Allowed; DDL lands in the branch schema |
| v0.4–v0.5 | Written straight to main, invisible in the branch diff | **Rejected** | Blocked |

**Version gate (v0.6+):** when `netbox_branching` is in `PLUGINS`, the plugin requires **NetBox ≥ 4.6.2** and **netbox-branching ≥ 1.0.4**; otherwise startup fails with system check `netbox_custom_objects.E001` / `E002`. Without branching installed, the normal compatibility matrix (NetBox 4.5.2–4.7.x) applies. Install with `pip install "netboxlabs-netbox-custom-objects[branching]"` to pull the branching floor. On NetBox 4.7 use netbox-branching 1.2.x; on 4.4–4.6 use 1.1.x (see [netbox-branching](../netbox-branching/SKILL.md)).

**`exempt_models` warning:** the published branching doc page still shows the pre-0.6 `PLUGINS_CONFIG['netbox_branching']['exempt_models']` snippet (exempting `customobjecttype` / `customobjecttypefield`) and the "instance writes are disallowed" text. The v0.6+ integration is tested with **no exemptions**. Exempting the schema models on v0.6+ routes type/field changes to main while instances live in the branch — do not carry the snippet forward unless the release notes for your version require it. Verify against the running version.

More: [references/graphql-and-branching.md](references/graphql-and-branching.md).

## Config Context and Config Templates

- **Config context** *(v0.6+)*: create the COT with `config_context_enabled: true`. Instances gain `local_context_data` (REST-writable JSON) and a Config Context tab. Source ConfigContexts are aggregated **by field-naming convention**: a non-polymorphic `object` field named `site`, `tenant`, `role`, `platform`, `location`, `device_type`, or `cluster` pointing at the matching model feeds that dimension. A type with none of these fields gets only its local data (global contexts are *not* applied).
- **Config templates** *(v0.6.1+, **NetBox 4.7+**)*: reference custom objects from device/VM config templates by the COT's **internal `name`** (not slug):

```jinja2
{% for iface in custom_objects.ospf_interface.filter(device=device) %}
interface {{ iface.name }}
 ip ospf area {{ iface.area }}
{% endfor %}
{# equivalent filter form: #}
{% for peer in 'bgp_peer' | custom_objects %}...{% endfor %}
{# names starting with a digit: #} {% for o in custom_objects['123foo'].all() %}...{% endfor %}
```

Unknown names log a warning and render as an empty, chainable stand-in (no rows, no error). On NetBox < 4.7 the `custom_objects` variable is simply absent.

## Portable Schema (v0.5+)

Export COT definitions as YAML/JSON, version them, and apply to other instances. `POST schema/preview/` returns a diff (`is_new`, `has_destructive_changes`, `field_changes[]`); `POST schema/apply/` with `{"allow_destructive": false, "schema": {...}}` applies atomically, returning 409 `destructive_changes` when column drops are present and not allowed. Both accept `Content-Type: application/yaml` or `application/json` *(YAML v0.7+)* and return JSON unless `Accept: application/yaml`. Fields match by stable `schema_id`; deletions need a `removed_fields` tombstone. Full workflow: [references/api-workflow.md](references/api-workflow.md#portable-schema).

## Programmatic Access (Scripts & Plugins)

Documented pattern for custom scripts and plugins:

```python
from netbox_custom_objects.models import CustomObjectType

DHCPScope = CustomObjectType.objects.get(name="dhcp_scope").get_model()
scope = DHCPScope.objects.filter(scope_name="office-floor-2").first()
```

The returned class behaves like a standard Django model (filter, create, update, delete, annotate). With **pynetbox 7.8+**, register `CustomObjectsExtension` so JSON columns round-trip and per-type endpoints return typed records — example in [references/api-workflow.md](references/api-workflow.md#pynetbox).

## Inherited NetBox Features

Every custom object automatically gets: tags, bookmarks, journaling, change logging, contacts *(v0.6+)*, ownership *(v0.6+)*, list/detail views, import/export, global search (per-field `search_weight`), event rules and notifications, custom links, cloning, NetBox custom fields pointing at the type, REST filtering/pagination/ETags (NetBox 4.6+), and object permissions.

## Important Constraints

1. **Deletion is destructive**: deleting a COT drops the table; deleting a COTF drops the column. Irreversible.
2. **No bulk create/update/delete on the instance API** — POSTing an array fails; iterate per object (or use the UI bulk import/edit).
3. **Validation is enforced on API writes** *(v0.5+)*: `required`, `validation_regex`, `validation_minimum/maximum`, plus `CUSTOM_VALIDATORS` keyed `netbox_custom_objects.<cot-slug>`.
4. **Max types**: 50 by default (`max_custom_object_types`; `0`/`None` disables).
5. **Reserved field names** (32): `id`, `pk`, `tags`, `owner`, `contacts`, `local_context_data`, `created`, `last_updated`, `custom_object_type`, `custom_field_data`, `bookmarks`, `journal_entries`, `model`, `objects`, `save`, `delete`, `clean`, `clone`, `snapshot`, … — the API error lists them all.
6. **Uniqueness** cannot be enforced on `boolean`, `multiobject`, or `coordinates` fields.
7. **Name format**: COT and COTF names match `^[a-z0-9]+(_[a-z0-9]+)*$`.
8. **Immutable after creation**: `is_polymorphic` + allowed types; `config_context_enabled`; a field's type cannot change to/from multi-column types (`coordinates`, `url`).
9. **CSV import** populates only the URL value and cannot set `<name>_title` or coordinates backing columns.
10. **Minimum NetBox**: v0.5.x–v0.7.x require **NetBox 4.5.2+**; v0.6.1+ for 4.7.x. Branching adds the 4.6.2 / 1.0.4 floor.

## Anti-Patterns

| Anti-pattern | Why it fails | Do this instead |
|--------------|--------------|-----------------|
| `"type": "location"` for a lat/long field | Not a valid type; the value is `coordinates` (release notes use the word "location") | `"type": "coordinates"`, then write `<name>_latitude` + `<name>_longitude` together |
| `"default": "\"active\""` via REST | `default` is JSON; the escaped form stores a string containing quotes | `"default": "active"` (UI form is where you type the quoted literal) |
| POSTing a JSON array to `/custom-objects/<slug>/` | The endpoint does not accept bulk arrays | Loop per object; expect 201 each |
| Assuming instance writes on a branch are rejected | True only for v0.4–v0.5 | On v0.6+ they are tracked in the branch; check `/api/status/` plugin version |
| Copying the `exempt_models` snippet onto a v0.6+ install | Splits schema (main) from instances (branch); v0.6+ tests run without it | Leave COT/COTF unexempted unless your version's docs say otherwise |
| GraphQL `dhcp_scopes_list` / `dhcpScopesList` | Root fields are prefixed and snake_case | `custom_objects_dhcp_scopes_list` |
| Sending GraphQL mutations or `filters:` | Not in the 0.7.0 schema | REST for writes and server-side filters |
| `display_expression` that references a field by label or with a typo | Undefined names render empty; render errors fall back silently to the primary field; API does no syntax check | Reference field `name`s; guard with `{% if field %}`; test in the UI form |
| Config template `custom_objects.dhcp-scopes` | Resolved by internal `name`, and hyphens aren't valid Jinja | `custom_objects.dhcp_scope` (or `custom_objects['123foo']`) |
| PATCHing `config_context_enabled` | Create-only | Recreate the type, or leave as-is |
| `on_delete_behavior` on a `multiobject` field | Ignored | Only meaningful on `object` fields |
| Naming a field `owner`, `contacts`, or `local_context_data` | Reserved (v0.6+ added these) | Pick another name (`owning_team`, …) |
| `CUSTOM_VALIDATORS['netbox_custom_objects.dhcp_scope']` | Key is the **slug** | `netbox_custom_objects.dhcp-scopes` |
| Referencing another COT with `"model": "table3model"` | Internal name; removed in v0.5 | `"app_label": "custom-objects", "model": "<target-slug>"` |
| pynetbox `nb.plugins.custom_objects.my_type` for slug `my_type` | Attribute access converts `_` to `-` | `nb.plugins.custom_objects.endpoint("my_type")` |
| Relying on GraphiQL autocomplete after adding a field | Explorer caches the schema per page load | Click "Re-fetch GraphQL schema"; the server is already current |

## References

Load on demand:

- [references/field-type-guide.md](references/field-type-guide.md) — **Load when** configuring fields: per-type attributes, `coordinates`/`url` multi-column behavior, validation matrix, choice sets.
- [references/api-workflow.md](references/api-workflow.md) — **Load when** building an end-to-end REST integration: full DHCP-scope walkthrough, filtering, error table, portable schema preview/apply, pynetbox extension.
- [references/graphql-and-branching.md](references/graphql-and-branching.md) — **Load when** reading via GraphQL or running Custom Objects alongside netbox-branching: field naming, type mapping, unions, version gate, what is/isn't branch-aware, config context aggregation rules.
- [references/modeling-patterns.md](references/modeling-patterns.md) — **Load when** designing a new type: lookup tables, bridges, hierarchies, display expressions, config-context conventions.

## Related Skills

- [netbox-data-modeling](../netbox-data-modeling/SKILL.md) — Check whether a built-in model already fits before creating a custom object
- [netbox-api-integration](../netbox-api-integration/SKILL.md) — Auth, pagination, bulk patterns, pynetbox basics
- [netbox-branching](../netbox-branching/SKILL.md) — Branch lifecycle, version lines (1.1.x for NetBox ≤ 4.6, 1.2.x for 4.7)
- [netbox-plugin-development](../netbox-plugin-development/SKILL.md) — When custom objects aren't enough
- [netbox-custom-scripts](../netbox-custom-scripts/SKILL.md) — Using custom objects in scripts

## Resources

- [GitHub Repository](https://github.com/netboxlabs/netbox-custom-objects) (source-available, NetBox Limited Use License)
- [Release Notes](https://github.com/netboxlabs/netbox-custom-objects/blob/main/docs/releases.md)
- [Compatibility Matrix](https://github.com/netboxlabs/netbox-custom-objects/blob/main/COMPATIBILITY.md)
- [pynetbox Custom Objects extension](https://github.com/netbox-community/pynetbox/blob/main/docs/custom-objects.md)
