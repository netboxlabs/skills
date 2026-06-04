# Custom Field Types

All custom field types available in NetBox 4.6 (the same 13 types ship on 4.5) with validation options, API format, and use cases.

## Field Types

| Type | API Value Format | Validation | Best For |
|------|-----------------|------------|----------|
| **text** | `"string"` | regex pattern, min/max length | Short identifiers: asset tags, serial numbers, cost centers |
| **longtext** | `"string"` | min/max length | Notes, descriptions, multi-line data |
| **integer** | `123` | min/max value | Counts, port numbers, priorities |
| **decimal** | `12.34` | min/max value | Costs, measurements, coordinates |
| **boolean** | `true`/`false` | — | Feature flags. Consider a tag if it's cross-object |
| **date** | `"2026-04-18"` | min/max date | Warranty expiry, install date, review date |
| **datetime** | `"2026-04-18T12:00:00Z"` | — | Timestamps for events |
| **url** | `"https://..."` | URL format | Monitoring links, documentation URLs |
| **json** | `{...}` or `[...]` | JSON Schema (via validation) | Structured data that doesn't fit other types |
| **selection** | `"choice-value"` | CustomFieldChoiceSet | Single-select from defined options: environment, tier |
| **multiselect** | `["a", "b"]` | CustomFieldChoiceSet | Multi-select: supported protocols, compliance frameworks |
| **object** | `{"id": 42}` or `42` | Specific object type | FK-like reference to another NetBox object |
| **multiobject** | `[42, 43]` | Specific object type | Multiple references: backup devices, related circuits |

## API Usage

### Setting Custom Field Values

Custom field data is nested under `custom_fields` in the object payload:

```json
{
  "name": "router-01",
  "custom_fields": {
    "warranty_expiry": "2027-12-31",
    "environment": "production",
    "cost_center": "CC-4200",
    "monitoring_url": "https://grafana.example.com/d/router-01"
  }
}
```

### Filtering by Custom Fields

```
GET /api/dcim/devices/?cf_environment=production
GET /api/dcim/devices/?cf_cost_center=CC-4200
```

## Choice Sets

Selection and multi-selection fields require a **CustomFieldChoiceSet**:

```json
{
  "name": "Environment",
  "extra_choices": [
    ["production", "Production"],
    ["staging", "Staging"],
    ["development", "Development"],
    ["lab", "Lab"]
  ]
}
```

Choice sets can be reused across multiple custom fields.

**NetBox 4.6** adds optional per-choice **colors**: each `extra_choices` entry may carry a third element — `["production", "Production", "4caf50"]` — so selection/multiselect values render as colored badges. Two-element `[value, label]` pairs remain valid (no color).

## Configuration Options

| Option | Values | Purpose |
|--------|--------|---------|
| `group_name` | string | Group related fields in UI tabs |
| `ui_visible` | always / if-set / hidden | Control UI display |
| `ui_editable` | yes / no / hidden | Control UI editability |
| `is_cloneable` | boolean | Include when cloning objects |
| `weight` | integer | Display order within group |
| `required` | boolean | Make field mandatory |
| `default` | varies | Pre-populated value for new objects |
| `search_weight` | integer | Weight in global search (0 = excluded) |
| `validation_schema` *(4.6)* | JSON Schema object | Validate **json**-type field values against a JSON Schema (replaces ad-hoc validation) |

## Guidelines

- **Prefer object/multiobject** over text fields for cross-references
- **Use group_name** to organize related fields (e.g., "Financial", "Compliance")
- **Set ui_visible=if-set** for rarely-used fields to reduce clutter
- **Use selection over boolean** when you might add more options later
- **Don't create custom fields for data that belongs in custom/config contexts** — if the value is inherited or computed, use a config context
