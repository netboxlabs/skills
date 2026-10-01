# API Workflow

Complete end-to-end workflow for creating and managing custom objects via the REST API (plugin v0.7.x, NetBox 4.5.2–4.7). Auth, pagination, and rate-limit patterns are owned by [netbox-api-integration](../../netbox-api-integration/SKILL.md).

## Full Workflow: DHCP Scope Manager

### 1. Create a Choice Set (for select fields)

```http
POST /api/extras/custom-field-choice-sets/

{
  "name": "Scope Status",
  "extra_choices": [
    ["active", "Active"],
    ["reserved", "Reserved"],
    ["deprecated", "Deprecated"]
  ]
}
```

Response includes `id` (e.g., `5`). Choice-set colors render as badges in the custom object UI *(v0.5.2+)*.

### 2. Create the Custom Object Type

```http
POST /api/plugins/custom-objects/custom-object-types/

{
  "name": "dhcp_scope",
  "slug": "dhcp-scopes",
  "description": "DHCP address scopes for network management",
  "verbose_name": "DHCP Scope",
  "verbose_name_plural": "DHCP Scopes",
  "group_name": "IPAM Extensions",
  "version": "1.0.0",
  "display_expression": "{{ scope_name }}{% if scope_status %} [{{ scope_status }}]{% endif %}"
}
```

Response includes `id` (e.g., `1`), plus read-only `fields` (nested), `schema_document`, `table_model_name`, and `object_type_name` (the `app_label.model` label to use when assigning object permissions to this type). Add `"config_context_enabled": true` here if instances need `local_context_data` — it cannot be enabled later.

Look a type up later by slug: `GET /api/plugins/custom-objects/custom-object-types/?slug=dhcp-scopes` *(v0.5.2+)*.

### 3. Add Fields

Add fields one at a time. Each references the COT by `id`.

**Primary name field:**
```http
POST /api/plugins/custom-objects/custom-object-type-fields/

{
  "custom_object_type": 1,
  "name": "scope_name",
  "label": "Scope Name",
  "type": "text",
  "required": true,
  "unique": true,
  "primary": true,
  "group_name": "Identity",
  "weight": 100,
  "search_weight": 100
}
```

**Object reference field:**
```http
POST /api/plugins/custom-objects/custom-object-type-fields/

{
  "custom_object_type": 1,
  "name": "ip_range",
  "label": "IP Range",
  "type": "object",
  "app_label": "ipam",
  "model": "iprange",
  "required": true,
  "context": true,
  "related_name": "dhcp_scopes",
  "on_delete_behavior": "protect",
  "group_name": "Network",
  "weight": 100
}
```

**Select field:**
```http
POST /api/plugins/custom-objects/custom-object-type-fields/

{
  "custom_object_type": 1,
  "name": "scope_status",
  "label": "Status",
  "type": "select",
  "choice_set": 5,
  "required": true,
  "default": "active",
  "group_name": "Identity",
  "weight": 200
}
```

> `default` is stored as JSON. Over REST, send the value itself (`"active"`, `86400`, `true`, `["a","b"]`). Only the UI form expects a typed JSON literal with quotes.

**Numeric field with validation:**
```http
POST /api/plugins/custom-objects/custom-object-type-fields/

{
  "custom_object_type": 1,
  "name": "lease_time",
  "label": "Lease Time (seconds)",
  "type": "integer",
  "default": 86400,
  "validation_minimum": 300,
  "validation_maximum": 604800,
  "group_name": "Settings",
  "weight": 100
}
```

**URL with title and coordinates (v0.6+/v0.7+):**
```http
POST /api/plugins/custom-objects/custom-object-type-fields/
{"custom_object_type": 1, "name": "runbook", "label": "Runbook", "type": "url"}

POST /api/plugins/custom-objects/custom-object-type-fields/
{"custom_object_type": 1, "name": "location", "label": "Location", "type": "coordinates"}
```

### 4. Create Custom Object Instances

The instance endpoint uses the COT's **slug**:

```http
POST /api/plugins/custom-objects/dhcp-scopes/

{
  "scope_name": "office-floor-2",
  "ip_range": 1,
  "scope_status": "active",
  "lease_time": 43200,
  "runbook": "https://wiki.example.com/dhcp/floor-2",
  "runbook_title": "Floor 2 runbook",
  "location_latitude": 40.7128,
  "location_longitude": -74.0060,
  "tags": [{"name": "production"}],
  "owner": 3
}
```

Response:
```json
{
  "id": 1,
  "url": ".../api/plugins/custom-objects/dhcp-scopes/1/",
  "display": "office-floor-2 [Active]",
  "owner": {"id": 3, "url": "...", "display": "Network Team"},
  "_context": {"ip_range": "10.0.1.100-200/24"},
  "created": "2026-01-15T10:30:00Z",
  "last_updated": "2026-01-15T10:30:00Z",
  "tags": [{"id": 1, "url": "...", "display": "production", "name": "production", "slug": "production"}],
  "scope_name": "office-floor-2",
  "ip_range": {"id": 1, "url": "...", "display": "10.0.1.100-200/24"},
  "scope_status": "active",
  "lease_time": 43200,
  "runbook": "https://wiki.example.com/dhcp/floor-2",
  "runbook_title": "Floor 2 runbook",
  "location_latitude": "40.712800",
  "location_longitude": "-74.006000"
}
```

Notes:
- `owner` *(v0.6+)* follows NetBox ownership (4.5+): write a PK or nested `{"id": N}`; omit or `null` to leave unset. If the type has its own field named `owner`, that field wins and the NetBox owner is not exposed (`owner` is reserved for new fields).
- `_context` appears only when the type has fields with `context: true`.
- `local_context_data` appears (and is writable) only when the type was created with `config_context_enabled: true`.
- Field-level validation (`required`, `validation_regex`, `validation_minimum/maximum`) and `CUSTOM_VALIDATORS` (`netbox_custom_objects.dhcp-scopes`) are enforced on API writes *(v0.5+)* — expect 400 with per-field errors. Optional `object`/`multiobject` fields accept `null` *(v0.5.2+)*.
- Polymorphic values: `{"app_label": "dcim", "model": "device", "object_id": 7}` (or `{"content_type_id": 23, "id": 7}`); list of dicts for polymorphic `multiobject`.

### 5. Query and Filter

```http
GET /api/plugins/custom-objects/dhcp-scopes/?scope_status=active
GET /api/plugins/custom-objects/dhcp-scopes/?scope_name=office            # substring (filter_logic=loose)
GET /api/plugins/custom-objects/dhcp-scopes/?lease_time__gte=3600&lease_time__lt=86400
GET /api/plugins/custom-objects/dhcp-scopes/?ip_range=1                   # or ?ip_range_id=1
GET /api/plugins/custom-objects/dhcp-scopes/?id=1&id=2
GET /api/plugins/custom-objects/dhcp-scopes/?tag=production&q=office
GET /api/plugins/custom-objects/dhcp-scopes/?limit=10&offset=0
```

- `filter_logic` on the field controls text matching: `loose` (default, `icontains`), `exact` (adds NetBox suffix lookups such as `__ic`, `__isw`, `__n`), `disabled` (no filter).
- Polymorphic fields get one filter per allowed type: `?<field>_<app_label>_<model>=<pk>`.
- Lookup suffixes on numeric/date fields work from v0.6.1 (earlier versions silently ignored them, as they did `?id=`).
- Every instance list supports NetBox's standard `q`, `tag`, `created__gte`, `last_updated__gte`, `limit`/`offset`, and `ordering`. Check `/api/schema/swagger-ui/` for the generated filter set of a given type.

### 6. Update

```http
PATCH /api/plugins/custom-objects/dhcp-scopes/1/

{
  "lease_time": 86400,
  "scope_status": "reserved"
}
```

On NetBox 4.6+ the detail endpoint returns an `ETag`; send `If-Match` to guard against concurrent edits.

### 7. Reverse Lookups

Find custom objects that reference a specific IP range:

```http
GET /api/plugins/custom-objects/linked-objects/?object_type=ipam.iprange&object_id=1
```

### 8. Cleanup

```http
DELETE /api/plugins/custom-objects/dhcp-scopes/1/
DELETE /api/plugins/custom-objects/custom-object-type-fields/7/
DELETE /api/plugins/custom-objects/custom-object-types/1/
```

Delete instances first, then fields, then the type (reverse dependency order). Deleting an object referenced by another custom object's `object` field follows that field's `on_delete_behavior` (`set_null` default, `cascade`, `protect` → 409/validation error).

> **Warning:** Deleting a COT drops the entire database table. Deleting a COTF drops the column. Irreversible.

## Bulk Operations

Bulk create (posting an array), bulk update, and bulk delete are **not supported** on the instance endpoints — the viewset is per-object only. Loop with one request per object and handle each response. For large loads prefer the UI bulk import (CSV/YAML/JSON) — note CSV cannot populate `<url>_title` or coordinates backing columns.

## Modifying Field Definitions

```http
PATCH /api/plugins/custom-objects/custom-object-type-fields/1/

{
  "label": "Updated Label",
  "required": false
}
```

- **`unique` false → true** fails with a validation error if duplicates exist.
- **Renaming** (`name`) renames the column; data is preserved. `schema_id` stays the same, so portable-schema diffs see a rename, not delete+add.
- **Cannot change**: `is_polymorphic`, `related_object_types`, and the type of a multi-column field (`coordinates`, `url`) to or from a single-column type. Delete and recreate instead.
- **Deprecating**: set `deprecated: true` (+ `deprecated_since`, `scheduled_removal`) to make the field read-only in the UI during a grace period.

## Portable Schema

Move type definitions between environments (dev → staging → prod) without hand-replaying field POSTs. Documents are YAML (recommended) or JSON; the same structure either way.

```yaml
schema_version: "1"
types:
  - name: dhcp_scope
    slug: dhcp-scopes
    verbose_name: DHCP Scope
    verbose_name_plural: DHCP Scopes
    version: "1.1.0"
    fields:
      - { id: 1, name: scope_name, type: text, required: true, unique: true, primary: true }
      - { id: 2, name: ip_range, type: object, related_object_type: ipam/iprange, required: true }
      - { id: 3, name: scope_status, type: select, choice_set: Scope Status }
      - { id: 5, name: parent, type: object, related_object_type: custom-objects/dhcp-scopes }
    removed_fields:
      - { id: 4, name: legacy_code, type: text, removed_in: "1.1.0" }
```

Key rules:
- `id` is the field's stable `schema_id` (read-only in the field API, auto-assigned). Fields are matched **only** by it; never reuse an id.
- `related_object_type` is `app_label/model` or `custom-objects/<slug>`; `choice_set` is the choice set **name**.
- A field missing from `fields` without a tombstone in `removed_fields` produces a warning, not a removal.
- `comments` is intentionally excluded from the format.
- The plugin's own docs describe an export helper that runs inside the NetBox shell (`manage.py nbshell`) — see the portable-schema docs page; there is no REST export endpoint in v0.7.0.

**Preview** (never modifies the DB, never returns 409):

```http
POST /api/plugins/custom-objects/schema/preview/
Content-Type: application/yaml
Accept: application/yaml

<schema document>
```

Response: `diffs[]` with `slug`, `is_new`, `has_changes`, `has_destructive_changes`, `cot_changes`, `field_changes[] {op: add|remove|alter, schema_id, db_name, schema_def}`, `warnings[]`.

**Apply** (atomic; requires `add_customobjecttype` + `change_customobjecttype`, and a write-enabled token):

```http
POST /api/plugins/custom-objects/schema/apply/
Content-Type: application/json

{"allow_destructive": false, "schema": { ...document... }}
```

| Status | Meaning |
|--------|---------|
| 200 `{"applied": true, "diffs": [...]}` | Applied |
| 409 `{"error": "destructive_changes", "destructive_slugs": [...]}` | Document drops columns; re-send with `allow_destructive: true` after review |
| 400 `{"error": "circular_dependency" \| "unresolvable_reference", "detail": ...}` | Cycle among new types, unknown choice set / object type / field type, or invalid document |
| 403 | Missing COT add/change permission |

Both endpoints accept `Content-Type: application/yaml` or `application/json` *(YAML v0.7+)* and reply in JSON unless `Accept: application/yaml`. On v0.6+ apply works inside an active branch (DDL lands in the branch schema); on v0.5 it was blocked while a branch was active.

## pynetbox

pynetbox **7.8.0+** ships `CustomObjectsExtension`. Register it so JSON columns (`related_object_filter`, `schema_document`, `_context`) round-trip as dicts and per-type endpoints return typed records:

```python
import pynetbox
from pynetbox.extensions import CustomObjectsExtension
from pynetbox.extensions.custom_objects import schema_preview, schema_apply

nb = pynetbox.api(
    "https://netbox.example.com",
    token="nbt_abc123.xxxxxxxxxxxxxxxx",
    extensions=[CustomObjectsExtension],
)

# Types and fields
cot = nb.plugins.custom_objects.custom_object_types.get(slug="dhcp-scopes")
print(cot.fields[0].related_object_type)          # {"id": ..., "app_label": "ipam", "model": "iprange"}

# Instances — attribute access turns "_" into "-", matching slug "dhcp-scopes"
scope = nb.plugins.custom_objects.dhcp_scopes.create(scope_name="office-floor-3", ip_range=1, scope_status="active")
for s in nb.plugins.custom_objects.dhcp_scopes.filter(scope_status="active"):
    print(s.scope_name, s.custom_object_type.slug)

# Slug that really contains an underscore: pass it verbatim
nb.plugins.custom_objects.endpoint("dhcp_scope").all()

# Reverse lookups
for row in nb.plugins.custom_objects.linked_objects.filter(object_type="ipam.iprange", object_id=1):
    print(row.custom_object_type["slug"], row.field_name, row.object["id"])

# Portable schema
diff = schema_preview(nb, document)                       # {"diffs": [...]}
try:
    result = schema_apply(nb, document, allow_destructive=False)
except pynetbox.RequestError as exc:
    if exc.req.status_code == 409:                        # exc.error carries destructive_slugs
        result = schema_apply(nb, document, allow_destructive=True)
    else:
        raise
```

Without the extension, basic CRUD still works but JSON columns are mangled into nested `Record` objects.

## Error Handling

| Status | Cause | Example |
|--------|-------|---------|
| 400 | Reserved field name | `"name": "Field name \"tags\" is reserved and cannot be used. Reserved names are: ..."` |
| 400 | Circular reference | `"Circular reference detected. This field would create a circular dependency ..."` |
| 400 | Missing choice_set | `"choice_set": "Selection fields must specify a set of choices."` |
| 400 | Uniqueness violation on toggle | `"unique": "Custom objects with non-unique values already exist so this action isn't permitted"` |
| 400 | `unique`/`default` on a type that forbids it | `"Uniqueness cannot be enforced for boolean or multiobject fields"` / `"... coordinates fields"` |
| 400 | Invalid default | `"default": "Invalid default value \"x\": ..."` |
| 400 | Invalid COT slug in cross-COT reference | `"Invalid custom object type slug."` |
| 400 | Max types exceeded | `"Maximum number of Custom Object Types (50) exceeded; adjust max_custom_object_types to raise this limit"` |
| 400 | `config_context_enabled` changed after creation | `"Config context support cannot be changed after creation."` |
| 400 | Coordinates half-set | Latitude and longitude must both be set or both empty |
| 403 | Schema apply without COT add+change perms | `"You do not have permission to apply a schema document. ..."` |
| 409 | Destructive schema apply without `allow_destructive` | `{"error": "destructive_changes", ...}` |
| 409 / 400 | Protected reference | Deleting an object referenced by a field with `on_delete_behavior: protect` |
