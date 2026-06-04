# Branch API Patterns

Base URL: `https://netbox.example.com`
All examples use `Authorization: Bearer <token>`.

## Create a Branch

```http
POST /api/plugins/branching/branches/
Content-Type: application/json

{
  "name": "add-london-site",
  "description": "Add London site and related infrastructure"
}
```

Response (201):
```json
{
  "id": 7,
  "url": "https://netbox.example.com/api/plugins/branching/branches/7/",
  "name": "add-london-site",
  "status": {"value": "new", "label": "New"},
  "schema_id": "a1b2c3d4",
  "owner": {"id": 1, "username": "admin"},
  "last_sync": null,
  "created": "2025-01-15T10:00:00Z"
}
```

Auto-provisions immediately. Poll status until `ready`.

## Poll Branch Status

```http
GET /api/plugins/branching/branches/7/
```

Wait for `"status": {"value": "ready"}`. Exponential backoff: 1s → 2s → 4s → max 30s.

## Activate Branch Context (Header)

Add to any standard NetBox API request:

```
X-NetBox-Branch: a1b2c3d4
```

The value is **`schema_id`**, not name or numeric ID. Branch must be `ready`.

### Example — Create Object in Branch

```http
POST /api/dcim/sites/
X-NetBox-Branch: a1b2c3d4
Content-Type: application/json

{"name": "London", "slug": "london", "status": "planned"}
```

### Example — Read from Branch

```http
GET /api/dcim/sites/?name=London
X-NetBox-Branch: a1b2c3d4
```

### Example — GraphQL in Branch

```http
POST /api/graphql/
X-NetBox-Branch: a1b2c3d4
Content-Type: application/json

{"query": "{ site_list { name status } }"}
```

## Review Changes (ChangeDiffs)

```http
GET /api/plugins/branching/changes/?branch_id=7
```

Returns list of `ChangeDiff` objects:
```json
{
  "id": 1,
  "branch": {"id": 7, "name": "add-london-site"},
  "object_type": "dcim.site",
  "object_id": 42,
  "object_repr": "London",
  "action": {"value": "create", "label": "Created"},
  "original_data": null,
  "modified_data": {"name": "London", "slug": "london"},
  "current_data": {"name": "London", "slug": "london", "status": "active", ...},
  "conflicts": null
}
```

## Sync from Main

```http
POST /api/plugins/branching/branches/7/sync/
Content-Type: application/json

{"commit": true}
```

Response (200): Job object with `url` for polling.

**Dry-run** (validate without applying):
```json
{"commit": false}
```

### If Conflicts Exist (409)

```json
{
  "detail": "All conflicts must be acknowledged before this branch can be synced.",
  "conflicts": [
    {
      "object_type": "dcim.device",
      "object_id": 42,
      "conflicts": ["name"],
      "conflicting_data": {
        "original": {"name": "switch-01"},
        "branch": {"name": "switch-london-01"},
        "main": {"name": "switch-nyc-01"}
      }
    }
  ]
}
```

Re-submit with acknowledgment:
```json
{"commit": true, "acknowledge_conflicts": true}
```

Branch values win on acknowledged conflicts.

## Merge to Main

```http
POST /api/plugins/branching/branches/7/merge/
Content-Type: application/json

{"commit": true}
```

Same async pattern — returns Job, poll for completion. Branch becomes `merged` on success.

Same conflict handling as sync: 409 if unacknowledged conflicts → add `"acknowledge_conflicts": true`.

**Merge is all-or-nothing.** Any validation failure rolls back the entire transaction.

## Revert a Merged Branch

```http
POST /api/plugins/branching/branches/7/revert/
Content-Type: application/json

{"commit": true}
```

Only valid for `merged` branches. Returns branch to `ready` state. Async job.

## Archive a Branch

```http
POST /api/plugins/branching/branches/7/archive/
```

Drops the PostgreSQL schema. Branch metadata retained. Terminal state.

Valid from `merged` or `ready` states.

## Delete a Branch

```http
DELETE /api/plugins/branching/branches/7/
```

Deprovisions (drops schema) then deletes the record entirely. Cannot delete the currently active branch.

## Branch Events

```http
GET /api/plugins/branching/branch-events/?branch_id=7
```

Read-only audit log of branch lifecycle events.

## Discover Branchable Models

```http
GET /api/plugins/branching/branchable-models/
```

Returns all models that support branching, with `app_label` and `model` fields. Available across the supported 4.4.1+ / plugin 1.0.x range (older 0.8.x builds returned 404). See [branch-aware-models.md](branch-aware-models.md) for the practical heuristics.

## Required Permissions

| Action | Permission |
|--------|-----------|
| Sync | `netbox_branching.sync_branch` |
| Merge | `netbox_branching.merge_branch` |
| Migrate (v1.0+) | `netbox_branching.migrate_branch` |
| Revert | `netbox_branching.revert_branch` |
| Archive | `netbox_branching.archive_branch` |

Standard CRUD on branches uses `netbox_branching.add_branch`, `change_branch`, `delete_branch`, `view_branch`.
