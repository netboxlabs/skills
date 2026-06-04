---
name: netbox-custom-objects
description: >
  No-code data model extensibility for NetBox. Create custom object types with typed fields,
  relationships to core models and other custom objects, full REST API, and standard NetBox
  features (tags, change logging, search, bookmarks, journaling).
license: Apache-2.0
---

# NetBox Custom Objects

> **Your knowledge of Custom Objects may be outdated.** Field types, relationship options, and API behavior evolve between plugin releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Custom Objects docs | `https://netboxlabs.com/docs/extensions/custom-objects/` | Overview, field types |
| Custom Objects API | `https://netboxlabs.com/docs/extensions/custom-objects/api/` | REST API reference |
| NetBox MCP server | If configured — list existing custom object types and instances | Schema discovery |
| NetBox Platform MCP | If configured — full CRUD on custom objects | Create types, manage instances |

## FIRST: Verify Connectivity

Confirm the Custom Objects plugin is installed:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/plugins/custom-objects/custom-object-types/" | python -m json.tool
```

If you get 404, the plugin is not installed. If 403, your token needs `custom_objects` permissions.

> **Version note:** this skill targets plugin **v0.5.x** (latest v0.5.1), which requires **NetBox 4.5.2+** (through 4.6.x). v0.5 changed several behaviors vs v0.4.x — cross-COT references, API validation, and branching — these are flagged inline below.

---

Extend the NetBox data model without writing code. Custom Objects let administrators define new object types with typed fields, validation rules, and relationships — all through the UI or API.

## When to Use This Skill

- Creating new object types in NetBox (e.g., DHCP Scopes, Contracts, Applications)
- Adding fields to custom object types (text, numeric, object references, etc.)
- Building integrations that create or consume custom objects via API
- Automating custom object type provisioning in scripts or CI/CD
- Querying linked objects (reverse lookups from core objects to custom objects)

## Quick Reference

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Custom Object Type (COT)** | A new model definition — like a database table. Has name, slug, fields. |
| **Custom Object Type Field (COTF)** | A column on a COT — typed, with validation, ordering, grouping. |
| **Custom Object** | An instance of a COT — like a row in the table. |
| **Primary field** | The COTF whose value becomes the object's display name. |
| **Group name** | Groups related COTs in the navigation sidebar. |

### API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/api/plugins/custom-objects/custom-object-types/` | CRUD on type definitions |
| `/api/plugins/custom-objects/custom-object-type-fields/` | CRUD on field definitions |
| `/api/plugins/custom-objects/{slug}/` | CRUD on instances (dynamic per type) |
| `/api/plugins/custom-objects/linked-objects/` | Reverse lookups from any object |

### Field Types (14)

| Type | Description | Validation Options |
|------|-------------|-------------------|
| `text` | Short text | `validation_regex` |
| `longtext` | Multi-line text (textarea) | `validation_regex` |
| `integer` | Whole number | `validation_minimum`, `validation_maximum` |
| `decimal` | Decimal number | `validation_minimum`, `validation_maximum` |
| `boolean` | True/false | — |
| `date` | ISO 8601 date (YYYY-MM-DD) | — |
| `datetime` | ISO 8601 datetime | — |
| `url` | URL string | `validation_regex` |
| `json` | Arbitrary JSON | — |
| `select` | Single choice from set | `choice_set` (required) |
| `multiselect` | Multiple choices from set | `choice_set` (required) |
| `object` | Reference to one object | `app_label`, `model`, `related_object_filter` |
| `multiobject` | Reference to many objects | `app_label`, `model`, `related_object_filter` |

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
  "group_name": "IPAM Extensions"
}
```

**Naming rules:**
- `name`: lowercase alphanumeric + underscores only. No double underscores. Must be unique.
- `slug`: URL-safe identifier. Used in API endpoint path.
- `group_name`: Optional. Groups this type with others in the sidebar navigation.

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
| `primary` | Use this field's value as the object's display name | Only one field per type should be primary |
| `required` | Field must have a value | Enforced on create and update |
| `unique` | Value must be unique across all instances | Cannot be set on `boolean` or `multiobject` fields |
| `weight` | Display ordering (lower = higher) | Default: 100 |
| `group_name` | Visual grouping in forms | Fields with same group_name appear together |
| `search_weight` | Search ranking (100=high, 500=medium, 1000=low, 0=excluded) | Default: 500 |
| `default` | Default value (JSON) | Strings must be double-quoted: `"\"default text\""` |
| `is_cloneable` | Include when cloning objects | Default: false |

### Step 3: Create Instances

```http
POST /api/plugins/custom-objects/dhcp-scopes/
```

```json
{
  "scope_name": "office-floor-2",
  "ip_range": 5,
  "vlan": 100
}
```

The endpoint path uses the COT's **slug** (not name).

## Object Reference Fields

Reference core NetBox objects or other custom objects using `object` and `multiobject` fields.

### Referencing Core Objects

```json
{
  "custom_object_type": 1,
  "name": "site",
  "label": "Site",
  "type": "object",
  "app_label": "dcim",
  "model": "site"
}
```

Common `app_label.model` combinations:
- `dcim.device`, `dcim.site`, `dcim.rack`, `dcim.interface`
- `ipam.prefix`, `ipam.ipaddress`, `ipam.vlan`, `ipam.iprange`
- `tenancy.tenant`, `circuits.circuit`, `virtualization.virtualmachine`

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

> **Plugin v0.5+**: the `"custom-objects"` + slug form is the supported way to reference another COT. (Older builds required an internal `table{id}model` name; do not use that against v0.5+.)

> **Self-referential fields** (a COT pointing to itself) are allowed. Circular references between different COTs are detected and blocked.

### Polymorphic Reference Fields

> **Plugin v0.5+**: an `object`/`multiobject` field can reference **multiple** object types. Set `is_polymorphic: true` and provide the allowed types via `related_object_types_input` instead of `app_label`/`model`:

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

When writing an instance value for a polymorphic field, pass a dict identifying the type and object: `{"app_label": "dcim", "model": "device", "object_id": 7}` (`content_type_id` and `id` are accepted aliases). The `is_polymorphic` flag and allowed types are immutable after the field is created.

### Object-Deletion Behavior

> **Plugin v0.5+**: `object` fields take an `on_delete_behavior` of `"set_null"` (default), `"cascade"`, or `"protect"`, controlling what happens to the custom object when a referenced object is deleted. Use `related_name` to set the reverse-accessor name for ORM lookups.

### Filtering Object Selections

Narrow the available choices using `related_object_filter`:

```json
{
  "custom_object_type": 1,
  "name": "active_device",
  "label": "Active Device",
  "type": "object",
  "app_label": "dcim",
  "model": "device",
  "related_object_filter": {"status": "active"}
}
```

## Linked Objects (Reverse Lookups)

Find all custom objects that reference a given core object:

```http
GET /api/plugins/custom-objects/linked-objects/?object_type=dcim.device&object_id=42
```

Response:

```json
{
  "count": 2,
  "results": [
    {
      "custom_object_type": {"id": 1, "name": "Server", "slug": "servers"},
      "field_name": "primary_device",
      "object": {"id": 7, "display": "web-server-01", "url": "..."}
    }
  ]
}
```

Both `object_type` and `object_id` are required parameters.

## Programmatic Access (Scripts & Plugins)

Use custom objects in NetBox custom scripts or plugins:

```python
from netbox_custom_objects.models import CustomObjectType

# Get the model class
cot = CustomObjectType.objects.get(name="dhcp_scope")
DHCPScope = cot.get_model()

# Query like any Django model
scopes = DHCPScope.objects.all()
scope = DHCPScope.objects.filter(scope_name="office-floor-2").first()
print(scope.scope_name, scope.site)
```

The generated model supports the full Django ORM (filter, create, update, delete, annotations, aggregations).

## Branching Compatibility

Custom Objects is compatible with NetBox Branching but with limitations. **The two layers behave differently** — don't conflate them:

- **Type and field definitions** (COT/COTF): writable while a branch is active, but changes apply **directly to main** — they do not appear in the branch diff or "Changes Ahead" view.
- **Custom object instances**: writes on a branch are **disallowed** (plugin v0.5+). In an active branch you can still read existing instances, but you **cannot create, edit, or delete** custom objects until the branch is merged/exited. (Do not assume instance writes silently land on main — they are rejected.)

To move COT/COTF schemas between instances or promote branch-developed schemas, use the **portable schema** export/apply feature rather than editing on a branch.

**Required configuration** when using with Branching:

```python
PLUGINS_CONFIG = {
    'netbox_branching': {
        'exempt_models': [
            'netbox_custom_objects.customobjecttype',
            'netbox_custom_objects.customobjecttypefield',
        ],
    },
}
```

## Inherited NetBox Features

Every custom object automatically gets:
- Tags, bookmarks, journaling, change logging
- List views, detail views, import/export
- Search (configurable weight per field)
- Event rules and notifications
- Custom links, cloning
- REST API with filtering and pagination
- Object permissions

## Important Constraints

1. **Deletion is destructive**: Deleting a COT drops the entire database table. Deleting a COTF drops the column. Both are irreversible.
2. **No GraphQL support** yet — REST API only.
3. **No bulk create** — the instance API does not accept arrays. Create instances one at a time.
4. **Field validation is enforced on API writes** (plugin v0.5+): `required`, `validation_regex`, `validation_minimum`, and `validation_maximum` are now applied on REST API writes as well as the UI. NetBox's `CUSTOM_VALIDATORS` setting is also honored — key it as `netbox_custom_objects.<cot-slug>`. (In v0.4.x these were UI-only; do not rely on that against v0.5+.)
5. **Max types**: Default limit of 50 custom object types (configurable via `max_custom_object_types`).
6. **Reserved field names**: ~30 names are reserved (e.g., `id`, `tags`, `created`, `last_updated`, `model`, `objects`, `pk`, `save`, `delete`). The API returns a clear error if you try to use one.
7. **Uniqueness constraints**: Cannot be enforced on `boolean` or `multiobject` fields.
8. **Name format**: COT names and COTF names must be lowercase alphanumeric with underscores. No double underscores.
9. **Cross-COT references use the COT slug** (plugin v0.5+): `app_label: "custom-objects"` + `model: "<target-slug>"`. Polymorphic fields reference multiple types via `related_object_types_input`.
10. **Minimum NetBox version**: plugin v0.5.x requires **NetBox 4.5.2+** (through 4.6.x). Installs on 4.4.x / 4.5.0 / 4.5.1 are not supported by current releases.

## Common Patterns

See reference files for detailed patterns:
- [references/modeling-patterns.md](references/modeling-patterns.md) — Design patterns for common use cases
- [references/field-type-guide.md](references/field-type-guide.md) — Detailed field configuration and validation
- [references/api-workflow.md](references/api-workflow.md) — Complete API workflow with examples

## Related Skills

- [netbox-data-modeling](../netbox-data-modeling/) — Core NetBox data model (use before creating custom objects to check if a built-in model already fits)
- [netbox-api-integration](../netbox-api-integration/) — REST API patterns that apply to custom object endpoints
- [netbox-plugin-development](../netbox-plugin-development/) — When custom objects aren't enough and you need a full plugin
- [netbox-branching](../netbox-branching/) — Branch compatibility considerations
- [netbox-custom-scripts](../netbox-custom-scripts/) — Using custom objects in scripts

## Resources

- [GitHub Repository](https://github.com/netboxlabs/netbox-custom-objects)
- [API Documentation](https://github.com/netboxlabs/netbox-custom-objects/blob/main/docs/api.md)
- [Compatibility Matrix](https://github.com/netboxlabs/netbox-custom-objects/blob/main/COMPATIBILITY.md)
