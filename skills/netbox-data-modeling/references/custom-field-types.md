# Custom Field Types

All custom field types available in NetBox 4.7 (the same 13 types ship on 4.5 and 4.6) with validation options, API format, and use cases.

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
| **url** | `"https://..."` | URL format; *(4.7)* scheme must be in `ALLOWED_URL_SCHEMES`, scheme-less values stored as `https://…` | Monitoring links, documentation URLs |
| **json** | `{...}` or `[...]` | JSON Schema (via validation) | Structured data that doesn't fit other types |
| **selection** | write `"choice-value"`; read *(4.7)* `{"value": "choice-value", "label": "Label"}`, *(4.5–4.6)* `"choice-value"` | CustomFieldChoiceSet | Single-select from defined options: environment, tier |
| **multiselect** | write `["a", "b"]`; read *(4.7)* `[{"value": "a", "label": "A"}, …]`, *(4.5–4.6)* `["a", "b"]` | CustomFieldChoiceSet | Multi-select: supported protocols, compliance frameworks |
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

### Reading Selection Values (4.7+)

Selection and multi-selection values are returned as `{value, label}` objects in REST and GraphQL, matching NetBox's built-in choice fields. Write the raw value, read `.value`:

```json
"custom_fields": {
  "environment": {"value": "production", "label": "Production"},
  "frameworks": [{"value": "pci", "label": "PCI DSS"}, {"value": "hipaa", "label": "HIPAA"}]
}
```

```python
env = device["custom_fields"]["environment"]
env = env["value"] if isinstance(env, dict) else env   # works on 4.5–4.7
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
| `nulls_first` *(4.7)* | boolean (default true) | Sort objects with no value before (true) or after (false) valued ones when ordering by the field |
| `status` *(4.7, read-only)* | active / provisioning / deleting | Lifecycle state — see below |

## Field Status and Background Provisioning (4.7+)

Creating a field **with a default value**, and deleting a field, rewrite stored data on every object of the assigned types. When those types hold more than `BULK_UPDATE_CHUNK_SIZE` objects in total, the work runs as a background job (needs `rqworker`):

- `status: provisioning` — default being written; the field is not live yet (omitted from `custom_fields` on objects until `active`)
- `status: deleting` — data being purged; the name stays reserved until the job finishes
- A field stuck mid-operation stays in that status; delete and recreate it, or requeue the job from Background Tasks
- Bulk imports: create custom fields **before** loading data and, on large tables, poll `GET /api/extras/custom-fields/<id>/` until `status` is `active` before writing values

## Guidelines

- **Prefer object/multiobject** over text fields for cross-references
- **Use group_name** to organize related fields (e.g., "Financial", "Compliance")
- **Set ui_visible=if-set** for rarely-used fields to reduce clutter
- **Use selection over boolean** when you might add more options later
- **Don't create custom fields for data that belongs in custom/config contexts** — if the value is inherited or computed, use a config context
