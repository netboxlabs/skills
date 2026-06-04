# Field Type Guide

Detailed configuration reference for each Custom Object Type Field type.

## Text Fields

### `text` — Short Text
- Max display: single-line input
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
{
  "name": "notes",
  "type": "longtext",
  "required": false
}
```

## Numeric Fields

### `integer` — Whole Number

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
- Cannot have `unique` constraint
- Default accepts `true` or `false`

```json
{
  "name": "is_redundant",
  "type": "boolean",
  "default": false
}
```

## Date/Time Fields

### `date` — ISO 8601 Date
Values must be `YYYY-MM-DD` format.

```json
{
  "name": "warranty_expiry",
  "type": "date"
}
```

### `datetime` — ISO 8601 DateTime
Values must be `YYYY-MM-DD HH:MM:SS` format.

```json
{
  "name": "last_audit",
  "type": "datetime"
}
```

## URL Field

### `url` — URL String
- Supports `validation_regex` for additional patterns

```json
{
  "name": "documentation_url",
  "type": "url"
}
```

## JSON Field

### `json` — Arbitrary JSON
Stores any valid JSON (object, array, string, number, boolean, null).

```json
{
  "name": "metadata",
  "type": "json",
  "default": {}
}
```

## Choice Fields

### `select` — Single Choice
Requires a `choice_set` (created via `/api/extras/custom-field-choice-sets/`).

```json
{
  "name": "environment",
  "type": "select",
  "choice_set": 5,
  "required": true
}
```

### `multiselect` — Multiple Choices
Same as `select` but allows multiple values.

```json
{
  "name": "supported_protocols",
  "type": "multiselect",
  "choice_set": 8
}
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
References one instance of any NetBox model or custom object.

```json
{
  "name": "primary_site",
  "type": "object",
  "app_label": "dcim",
  "model": "site",
  "required": true,
  "related_object_filter": {"status": "active"}
}
```

### `multiobject` — Multiple Object References
References multiple instances. Creates an M2M relationship.

```json
{
  "name": "backup_devices",
  "type": "multiobject",
  "app_label": "dcim",
  "model": "device"
}
```

- Cannot have `unique` constraint
- Values are passed as an array of IDs: `"backup_devices": [1, 5, 12]`
- `related_object_filter` narrows the selection in the UI

### Cross-Custom-Object References

> **Plugin v0.5+**: reference another custom object type with `app_label: "custom-objects"` and `model` set to the target COT's **slug**:

```json
{
  "name": "parent_record",
  "type": "object",
  "app_label": "custom-objects",
  "model": "dhcp-scopes"
}
```

For a field that may reference several object types, set `is_polymorphic: true` and list the allowed types in `related_object_types_input` instead of `app_label`/`model`. (Older 0.4.x builds required an internal `table{id}model` name; do not use that against v0.5+.)

## Validation Summary

| Field Type | `required` | `unique` | `validation_regex` | `validation_min/max` | `choice_set` | `app_label`+`model` |
|-----------|-----------|---------|-------------------|---------------------|-------------|---------------------|
| text | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| longtext | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| integer | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| decimal | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| boolean | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| date | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| datetime | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| url | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| json | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| select | ✅ | ✅ | ❌ | ❌ | ✅ (required) | ❌ |
| multiselect | ✅ | ✅ | ❌ | ❌ | ✅ (required) | ❌ |
| object | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ (required) |
| multiobject | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ (required) |
