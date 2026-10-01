# Field Type Guide

Detailed configuration reference for each Custom Object Type Field type (plugin v0.7.x). All payloads go to `POST /api/plugins/custom-objects/custom-object-type-fields/` and also need `"custom_object_type": <id>`.

## Common Attributes (all types)

| Attribute | Default | Notes |
|-----------|---------|-------|
| `name` | — | `^[a-z0-9]+(_[a-z0-9]+)*$`; unique within the type; not a reserved name |
| `label` | `name` | Human-readable |
| `description` | `""` | Help text in forms |
| `group_name` | `""` | Fields sharing a group render together |
| `required` | `false` | Enforced on API and UI writes |
| `unique` | `false` | Not for `boolean`, `multiobject`, `coordinates` |
| `primary` | `false` | Display-name source (unless the COT sets `display_expression`) |
| `context` | `false` | Shown next to the object when referenced; REST `_context` |
| `default` | `null` | JSON value — send it raw over REST; not for `coordinates` |
| `weight` | `100` | Higher = lower on the page |
| `search_weight` | `500` | `0` excludes from search; 100 high, 1000 low |
| `filter_logic` | `"loose"` | `loose` / `exact` / `disabled` (text-like fields) |
| `ui_visible` | `"always"` | `always` / `if-set` / `hidden` |
| `ui_editable` | `"yes"` | `yes` / `no` (read-only) / `hidden` (still API-writable) |
| `is_cloneable` | `false` | Copied on clone |
| `comments` | `""` | Markdown; excluded from portable schema |
| `deprecated`, `deprecated_since`, `scheduled_removal` | `false`, `""`, `""` | Grace-period lifecycle; PEP 440 versions |
| `schema_id` | auto | Read-only stable id for portable schema |

## Text Fields

### `text` — Short Text
- Supports: `validation_regex`, `default`, `unique`, `required`

```json
{
  "name": "hostname",
  "type": "text",
  "required": true,
  "unique": true,
  "validation_regex": "^[a-zA-Z0-9-]+$"
}
```

### `longtext` — Multi-Line Text
- Display: textarea widget
- Supports: `validation_regex`, `default`, `required`

```json
{"name": "notes", "type": "longtext", "required": false}
```

## Numeric Fields

### `integer` — Whole Number
64-bit range *(v0.6+; 32-bit before)*. `validation_minimum`/`validation_maximum` are also 64-bit.

```json
{
  "name": "port_count",
  "type": "integer",
  "validation_minimum": 1,
  "validation_maximum": 1000,
  "default": 48
}
```

### `decimal` — Decimal Number

```json
{
  "name": "power_draw_kw",
  "type": "decimal",
  "validation_minimum": 0,
  "validation_maximum": 100
}
```

## Boolean

### `boolean` — True/False
- Cannot have `unique`
- `default` accepts `true` or `false`

```json
{"name": "is_redundant", "type": "boolean", "default": false}
```

## Date/Time Fields

### `date` — ISO 8601 Date
Values are `YYYY-MM-DD`.

```json
{"name": "warranty_expiry", "type": "date"}
```

### `datetime` — ISO 8601 DateTime
Values are ISO 8601 (`2026-01-15T10:30:00Z`, as NetBox core returns them).

```json
{"name": "last_audit", "type": "datetime"}
```

## URL Field

### `url` — URL with optional link title
- Supports `validation_regex`, `unique`, `default` (all apply to the URL value)
- *(v0.7+)* Two backing columns: `<name>` (URL) and `<name>_title` (display text). REST exposes both flat; the title replaces the raw URL as link text in detail and list views. Title and URL are independent — either may be set alone.
- CSV import sets only the URL. Existing URL fields gain the title column on upgrade (`manage.py migrate`).
- GraphQL exposes only the URL string.

```json
{"name": "documentation_url", "type": "url"}
```

Instance write: `{"documentation_url": "https://wiki.example.com/x", "documentation_url_title": "Runbook"}`

## JSON Field

### `json` — Arbitrary JSON
Stores any valid JSON (object, array, string, number, boolean, null). REST filter is substring match on the serialized value.

```json
{"name": "metadata", "type": "json", "default": {}}
```

## Choice Fields

### `select` — Single Choice
Requires a `choice_set` (a NetBox Custom Field Choice Set id, created via `/api/extras/custom-field-choice-sets/`). Values are the choice **keys**; the UI shows labels and choice-set colors *(v0.5.2+)*.

```json
{"name": "environment", "type": "select", "choice_set": 5, "required": true, "default": "production"}
```

### `multiselect` — Multiple Choices
Same as `select` but values are a list of keys.

```json
{"name": "supported_protocols", "type": "multiselect", "choice_set": 8}
```

**Creating a choice set first:**

```http
POST /api/extras/custom-field-choice-sets/
```
```json
{
  "name": "Environments",
  "extra_choices": [
    ["production", "Production"],
    ["staging", "Staging"],
    ["development", "Development"]
  ]
}
```

## Object Reference Fields

### `object` — Single Object Reference
References one instance of any NetBox model, plugin model, or custom object type.

```json
{
  "name": "primary_site",
  "type": "object",
  "app_label": "dcim",
  "model": "site",
  "required": true,
  "related_object_filter": {"status": "active"},
  "related_name": "servers",
  "on_delete_behavior": "set_null"
}
```

| Attribute | Notes |
|-----------|-------|
| `app_label` + `model` | Write-only; required unless polymorphic. Response returns read-only `related_object_type` |
| `related_object_filter` | `query_params` dict narrowing the UI selector |
| `related_name` | Reverse accessor on the target (templates, ORM); unique per target type |
| `on_delete_behavior` | `set_null` (default) / `cascade` / `protect` — **`object` only** |

Instance value: integer PK (`"primary_site": 12`), or `null` when optional *(v0.5.2+)*.

### `multiobject` — Multiple Object References
References many instances of one type (or several types when polymorphic).

```json
{"name": "backup_devices", "type": "multiobject", "app_label": "dcim", "model": "device"}
```

- Cannot have `unique`; `on_delete_behavior` is ignored
- Instance value: array of PKs — `"backup_devices": [1, 5, 12]`
- `related_object_filter` and `related_name` work as for `object`
- *(v0.6+)* Form fields support Quick Add for creating the target inline

### Cross-Custom-Object References

Reference another custom object type with `app_label: "custom-objects"` and `model` set to the target COT's **slug** *(v0.5+; the old `table{id}model` name is gone)*:

```json
{"name": "parent_record", "type": "object", "app_label": "custom-objects", "model": "dhcp-scopes"}
```

Self-references are allowed; cycles between different types are rejected.

### Polymorphic References

Set `is_polymorphic: true` and list allowed types in `related_object_types_input` (write-only; response returns `related_object_types`). Immutable after creation.

```json
{
  "name": "attached_to",
  "type": "object",
  "is_polymorphic": true,
  "related_object_types_input": [
    {"app_label": "dcim", "model": "device"},
    {"app_label": "virtualization", "model": "virtualmachine"},
    {"app_label": "custom-objects", "model": "appliances"}
  ],
  "related_name": "attachments"
}
```

Instance value: `{"app_label": "dcim", "model": "device", "object_id": 7}` (aliases: `content_type_id`, `id`); list of dicts for `multiobject`. REST filters: `?attached_to_dcim_device=7`. GraphQL: a union — select per type with inline fragments.

## Coordinates Field (v0.6+)

### `coordinates` — Latitude/Longitude Pair
One field, two backing columns, mirroring NetBox Site/Device coordinates (no PostGIS).

```json
{"name": "location", "type": "coordinates"}
```

- REST exposes `location_latitude` (−90…90) and `location_longitude` (−180…180), up to 6 decimal places, as two flat fields — **both or neither**; setting one is a 400.
- No `unique`, no `default`; cannot be converted to/from other types; CSV import cannot populate it.
- Detail view shows a Map button using NetBox's `MAPS_URL`.
- Not exposed in GraphQL.
- The API `type` value is `coordinates` even though release notes describe it as a "location" field type.

## Validation Summary

| Field Type | `required` | `unique` | `validation_regex` | `validation_min/max` | `choice_set` | `app_label`+`model` / polymorphic | `default` |
|-----------|-----------|---------|-------------------|---------------------|-------------|-----------------------------------|-----------|
| text | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| longtext | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| integer | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ |
| decimal | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ |
| boolean | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| date | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| datetime | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| url | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (URL only) |
| json | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| select | ✅ | ✅ | ❌ | ❌ | ✅ (required) | ❌ | ✅ |
| multiselect | ✅ | ✅ | ❌ | ❌ | ✅ (required) | ❌ | ✅ |
| object | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (required) | ✅ |
| multiobject | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ (required) | ✅ |
| coordinates | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

## GraphQL Type Mapping (v0.6+)

| Field type | GraphQL |
|------------|---------|
| text, longtext, url, select | `String` |
| integer | `Int` |
| decimal | `Decimal` |
| boolean | `Boolean` |
| date / datetime | `Date` / `DateTime` |
| json | `JSON` |
| multiselect | `[String]` |
| object / multiobject | target's native type / list; union when polymorphic; `CustomObjectRelatedObjectType` stub (`id`, `object_type`, `display`, `url`) for targets without a GraphQL type |
| coordinates | not exposed |
