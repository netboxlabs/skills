# Conflict Resolution

## What Causes Conflicts

A conflict occurs when the **same field** on the **same object** is modified in both main and the branch after the branch was last synced. The system detects this via `ChangeDiff` records by comparing three versions:

- **Original** — field value when the branch was created (or last synced)
- **Branch** — current value in the branch
- **Main** — current value in main

If both branch and main changed a field from its original value, that field is in conflict.

## How Conflicts Surface

### On ChangeDiff Records

```http
GET /api/plugins/branching/changes/?branch_id=7
```

Each `ChangeDiff` with conflicts has:

```json
{
  "object_type": "dcim.device",
  "object_id": 42,
  "object_repr": "switch-01",
  "action": {"value": "update", "label": "Updated"},
  "conflicts": {
    "name": {
      "original": "switch-01",
      "branch": "switch-london-01",
      "main": "switch-nyc-01"
    },
    "status": {
      "original": "active",
      "branch": "planned",
      "main": "staged"
    }
  }
}
```

### On Sync/Merge Attempts (HTTP 409)

When you attempt to sync or merge a branch with unacknowledged conflicts:

```http
POST /api/plugins/branching/branches/7/merge/
{"commit": true}
```

Response (409):
```json
{
  "detail": "All conflicts must be acknowledged before this branch can be merged.",
  "conflicts": [
    {
      "id": 1,
      "object_type": "dcim.device",
      "object_id": 42,
      "object_repr": "switch-01",
      "action": "update",
      "conflicts": ["name", "status"],
      "conflicting_data": {
        "original": {"name": "switch-01", "status": "active"},
        "branch": {"name": "switch-london-01", "status": "planned"},
        "main": {"name": "switch-nyc-01", "status": "staged"}
      }
    }
  ]
}
```

## Resolving Conflicts

### Option 1: Acknowledge and Proceed (Branch Wins)

Re-submit the sync/merge with acknowledgment:

```json
{"commit": true, "acknowledge_conflicts": true}
```

**Branch values win** on all acknowledged conflicts. Main's conflicting changes are overwritten.

### Option 2: Manually Resolve Before Merge

1. Review conflicts via the ChangeDiff API
2. Update the conflicting objects in the branch to desired values
3. Or sync first (to pull main's changes), then adjust in branch
4. Retry merge — if conflicts are resolved, no 409

### Option 3: Dry-Run First

```json
{"commit": false}
```

Validates the merge/sync without applying. Returns conflicts in the 409 response if any exist. Use this to preview before committing.

## Conflict Types by Action

| Branch Action | Main Action | Result |
|--------------|-------------|--------|
| Update field X | Update field X | **Conflict** on field X |
| Update field X | Update field Y | No conflict (different fields) |
| Update object | Delete object | Conflict — branch update skipped (squash) or fails (iterative) |
| Delete object | Update object | Branch delete wins |
| Create object | — | No conflict possible (new object) |

## Best Practices

1. **Sync frequently** to minimize conflict surface area.
2. **Dry-run before merge** to discover conflicts early.
3. **Review the 409 `conflicts` array** carefully — understand what branch vs main changed before acknowledging.
4. **Use squash merge** for complex branches — it handles CREATE+DELETE optimization and dependency ordering, reducing merge failures.
5. **Coordinate with Change Request workflow** — review conflicts as part of CR approval.
