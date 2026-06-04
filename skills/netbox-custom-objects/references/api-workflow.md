# API Workflow

Complete end-to-end workflow for creating and managing custom objects via the REST API.

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

Response includes `id` (e.g., `5`).

### 2. Create the Custom Object Type

```http
POST /api/plugins/custom-objects/custom-object-types/

{
  "name": "dhcp_scope",
  "slug": "dhcp-scopes",
  "description": "DHCP address scopes for network management",
  "verbose_name": "DHCP Scope",
  "verbose_name_plural": "DHCP Scopes",
  "group_name": "IPAM Extensions"
}
```

Response includes `id` (e.g., `1`).

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
  "default": "\"active\"",
  "group_name": "Identity",
  "weight": 200
}
```

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

### 4. Create Custom Object Instances

The instance endpoint uses the COT's **slug**:

```http
POST /api/plugins/custom-objects/dhcp-scopes/

{
  "scope_name": "office-floor-2",
  "ip_range": 1,
  "scope_status": "active",
  "lease_time": 43200,
  "tags": [{"name": "production"}]
}
```

Response:
```json
{
  "id": 1,
  "url": ".../api/plugins/custom-objects/dhcp-scopes/1/",
  "custom_object_type": {"id": 1, "name": "dhcp_scope", "description": "..."},
  "scope_name": "office-floor-2",
  "ip_range": {"id": 1, "url": "...", "display": "10.0.1.100-200/24", ...},
  "scope_status": "active",
  "lease_time": 43200,
  "tags": [{"id": 1, "url": "...", "display": "production", ...}],
  "created": "2025-01-15T10:30:00Z",
  "last_updated": "2025-01-15T10:30:00Z"
}
```

> **Plugin v0.5+:** Field-level validation (`required`, `validation_regex`, `validation_minimum/maximum`) is enforced on **REST API writes** as well as in the UI, and NetBox's `CUSTOM_VALIDATORS` (keyed `netbox_custom_objects.<cot-slug>`) is honored. Expect a 400 with field errors on invalid input. (In 0.4.x these were UI-only — do not assume the API skips validation.)

### 5. Query and Filter

```http
GET /api/plugins/custom-objects/dhcp-scopes/?scope_status=active
GET /api/plugins/custom-objects/dhcp-scopes/?scope_name__ic=office
GET /api/plugins/custom-objects/dhcp-scopes/?tag=production
GET /api/plugins/custom-objects/dhcp-scopes/?limit=10&offset=0
```

### 6. Update

```http
PATCH /api/plugins/custom-objects/dhcp-scopes/1/

{
  "lease_time": 86400,
  "scope_status": "reserved"
}
```

### 7. Reverse Lookups

Find custom objects that reference a specific IP range:

```http
GET /api/plugins/custom-objects/linked-objects/?object_type=ipam.iprange&object_id=1
```

### 8. Cleanup

```http
DELETE /api/plugins/custom-objects/dhcp-scopes/1/
```

Delete instances first, then fields, then the type (in reverse dependency order).

> **Warning:** Deleting a COT drops the entire database table. Deleting a COTF drops the column. These operations are irreversible.

## Bulk Operations

> **Note:** Bulk create (posting an array) is **not supported** for custom object instances — the API returns "Expected a dictionary, but got list." Create instances one at a time. Bulk delete is also not available via the plugin API.

## Modifying Field Definitions

Fields can be updated after creation:

```http
PATCH /api/plugins/custom-objects/custom-object-type-fields/1/

{
  "label": "Updated Label",
  "required": false
}
```

**Changing `unique` from false to true** will fail if existing data contains duplicates — the API returns a validation error explaining this.

**Renaming a field** (`name`) updates the database column. Existing data is preserved.

## Error Handling

Common errors:

| Status | Cause | Example |
|--------|-------|---------|
| 400 | Reserved field name | `"name": "Field name 'tags' is reserved..."` |
| 400 | Circular reference | `"related_object_type": "Circular reference detected..."` |
| 400 | Missing choice_set | `"choice_set": "Selection fields must specify a set of choices."` |
| 400 | Uniqueness violation | `"unique": "Custom objects with non-unique values already exist..."` |
| 400 | Max types exceeded | `"Maximum number of Custom Object Types (50) exceeded..."` |
| 409 | Foreign key constraint | Cannot delete a COT or object with dependent references |
