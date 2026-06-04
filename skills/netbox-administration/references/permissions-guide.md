# Permissions & Tokens Guide

## Object-Based Permissions

NetBox replaces Django's built-in permission system with object-level permissions. Each `ObjectPermission` consists of:

- **Object types** — ContentTypes the permission applies to
- **Actions** — `view`, `add`, `change`, `delete`, plus custom (e.g., `run` for scripts)
- **Constraints** — JSON filters restricting which specific objects
- **Users/Groups** — who receives the permission

### Constraint Syntax

Constraints use Django ORM field lookups in JSON:

```json
// AND: single dict — all conditions must match
{"status": "active", "region__name": "US"}

// OR: list of dicts — any set can match
[{"vid__gte": 100}, {"status": "reserved"}]

// Current user reference
{"created_by": "$user"}

// Supported lookups
{"name__startswith": "NYC"}
{"vid__in": [100, 200, 300]}
{"tenant__isnull": true}
{"name__iendswith": ".com"}
{"rack_id__gte": 1, "rack_id__lt": 100}
```

### How Constraints Are Enforced

- **View**: Queryset is filtered — users only see matching objects
- **Add/Change**: Object is saved in an atomic transaction, then re-queried with constraints. If not found → rollback with permission denied
- **Superusers** bypass all permission checks
- Multiple permissions on the same object type are merged with OR

### EXEMPT_VIEW_PERMISSIONS

```python
# Allow unauthenticated viewing of specific models
EXEMPT_VIEW_PERMISSIONS = ['dcim.device', 'ipam.ipaddress']

# Allow viewing most models (excludes sensitive ones like user permissions)
EXEMPT_VIEW_PERMISSIONS = ['*']
```

The wildcard `['*']` does **not** exempt sensitive models — you must list those explicitly if needed.

### DEFAULT_PERMISSIONS

Applied to **all** authenticated users regardless of explicit assignments.

Default value grants token self-management:
```python
{
    'users.view_token': {'user': '$user'},
    'users.add_token': {'user': '$user'},
    'users.change_token': {'user': '$user'},
    'users.delete_token': {'user': '$user'},
}
```

> **⚠️ Critical**: Setting custom `DEFAULT_PERMISSIONS` **completely replaces** the defaults. If you want to add permissions while keeping token self-management, you must include the token permissions in your custom dict.

## API Tokens

### v1 Tokens (Legacy, pre-4.5)

- 40-character plaintext stored in database
- Header: `Authorization: Token <plaintext>`
- **Deprecated in 4.6, removed in v5.0** — migrate to v2 before upgrading past 4.x

### v2 Tokens (4.5+)

- Key (short ID) + HMAC-SHA256 digest stored; plaintext **never** stored
- Header: `Authorization: Bearer nbt_<key>.<token>`
- Requires `API_TOKEN_PEPPERS` in configuration
- On **4.6.1+** the plaintext token is returned once, in the creation response — capture it then; it cannot be retrieved later

### Token Features

| Field | Purpose |
|-------|---------|
| `enabled` | Soft disable without deleting |
| `write_enabled` | `False` = read-only token |
| `allowed_ips` | IP/subnet restriction list |
| `expires` | Expiration datetime |
| `description` | Human-friendly label |

### Pepper Rotation (v2)

```python
API_TOKEN_PEPPERS = {
    1: 'original-pepper-string-at-least-50-chars...',
    2: 'new-pepper-string-at-least-50-chars...',  # Add new ID
}
```

New tokens use the highest-numbered pepper. Existing tokens continue validating with their original pepper. Never remove old peppers while tokens using them exist.

## Permission Design Patterns

### Team-Based Access

Create groups per team, assign permissions with site/region constraints:
```json
// "NYC Team" can manage devices in NYC
{"site__name": "NYC-DC1"}
```

### Read-Only Integration Accounts

1. Create a user for the integration
2. Assign `view` permission on needed object types
3. Issue a token with `write_enabled=False`
4. Restrict `allowed_ips` to the integration server

### Self-Service Model

Use `$user` to let users manage their own objects:
```json
{"created_by": "$user"}
```
