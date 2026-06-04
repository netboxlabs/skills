---
name: netbox-branching
description: >
  NetBox Branching plugin — create isolated branch schemas for safe change staging,
  review, and merge. Use when working with branch lifecycle, the X-NetBox-Branch header,
  async provisioning/sync/merge jobs, conflict resolution, or Change Request integration.
license: Apache-2.0
---

# NetBox Branching

Branching gives NetBox git-like change isolation. Each branch uses an **isolated PostgreSQL schema** — you read and write against it via a special header, then merge changes back to main.

> **Your knowledge of NetBox Branching may be outdated.** Branch states, merge strategies, and API behavior evolve between plugin releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Branching docs | `https://netboxlabs.com/docs/extensions/branching/` | Full reference, lifecycle, config |
| Branching REST API | `https://netboxlabs.com/docs/extensions/branching/rest-api/` | Endpoint details |
| Branching repo | `https://github.com/netboxlabs/netbox-branching` | Source, changelog, issues |
| NetBox MCP server | If configured — verify branch states, list branches | Live branch status |
| NetBox Platform MCP | If configured — full branch lifecycle operations | Create, sync, merge verification |

## FIRST: Verify Connectivity

Confirm the Branching plugin is installed and your token has access:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/plugins/branching/branches/" | python -m json.tool
```

You should see a paginated list of branches (or empty results). If you get 404, the Branching plugin is not installed. If 403, your token lacks `netbox_branching` permissions.

---

**Plugin:** `netbox_branching` 1.0.x (latest v1.0.3) · **NetBox:** 4.4.1–4.6

> **Plugin v1.0+** added a dedicated **`migrate`** branch action/permission (for applying pending migrations after a NetBox upgrade) and ships the `branchable-models` discovery endpoint on all supported versions. Permission codenames, the merge `squash` strategy, and the 11-state lifecycle are stable across the 1.0.x line.

## Core Concepts

- **Branch = isolated schema.** Created on provision, dropped on archive/delete. Each branch has its own database tables.
- **`schema_id`** (8-char alphanumeric) is the branch identifier for API access — not the name, not the numeric ID.
- **All heavy operations are async** — provision, sync, merge, revert return a Job. Poll until complete.
- **Conflicts** arise when main and branch modify the same object fields. Must be acknowledged before merge/sync proceeds.

## Branch Lifecycle

11 states. See [references/branch-lifecycle.md](references/branch-lifecycle.md) for the full state machine.

| State | Meaning |
|-------|---------|
| `new` | Created, not yet provisioned |
| `provisioning` | Schema being copied (async) |
| `ready` | Usable — read/write allowed |
| `syncing` | Pulling main→branch (async) |
| `migrating` | Applying DB migrations (async) |
| `merging` | Pushing branch→main (async) |
| `reverting` | Undoing a merge (async) |
| `merged` | Successfully merged; read-only |
| `archived` | Schema dropped, metadata kept |
| `pending-migrations` | Needs migration after NetBox upgrade |
| `failed` | Provisioning failed; terminal |

**Transitional states** (`provisioning`, `syncing`, `migrating`, `merging`, `reverting`) cannot be interrupted. On failure they revert to the previous stable state, except `provisioning` → `failed`.

## Quick Reference — Common Workflows

### Create and Wait for Ready

```http
POST /api/plugins/branching/branches/
Content-Type: application/json
Authorization: Bearer <token>

{"name": "add-new-site", "description": "Adding London site"}
```

Response includes `schema_id`. Auto-provisions. Poll until ready:

```http
GET /api/plugins/branching/branches/<id>/
```

Check `status` field — wait for `ready`. Typically seconds to minutes depending on DB size.

### Activate Branch Context

**For all API/GraphQL requests in branch context**, add the header:

```
X-NetBox-Branch: <schema_id>
```

The value is the **`schema_id`** (e.g., `a1b2c3d4`), NOT the branch name or ID.

Other methods (UI only, not for API automation):
- Cookie: `active_branch=<schema_id>`
- Query param: `?_branch=<schema_id>`

Branch must be in `ready` state or the request returns 400.

### Make Changes in Branch

Use normal NetBox API endpoints with the branch header:

```http
POST /api/dcim/sites/
X-NetBox-Branch: a1b2c3d4
Content-Type: application/json

{"name": "London", "slug": "london", "status": "planned"}
```

Changes are isolated to the branch schema. Main is unaffected.

### Review Changes (ChangeDiffs)

```http
GET /api/plugins/branching/changes/?branch_id=<id>
```

Returns `ChangeDiff` records showing what changed — object type, action (create/update/delete), original vs modified data.

### Sync from Main

Pull latest main changes into the branch:

```http
POST /api/plugins/branching/branches/<id>/sync/
Content-Type: application/json

{"commit": true}
```

Returns a Job object. Poll the job URL for completion. Sync may cascade-delete branch-only child objects if their parent was deleted in main.

**Dry-run:** `{"commit": false}` — validates without applying.

### Merge to Main

```http
POST /api/plugins/branching/branches/<id>/merge/
Content-Type: application/json

{"commit": true}
```

Returns a Job. On success, branch status becomes `merged` (read-only).

**Merge strategies:**
- **Iterative** (default) — applies/reverts changes one-at-a-time chronologically
- **Squash** — collapses to one operation per object, with dependency ordering. Handles bidirectional FK cycles. CREATE+DELETE = skip.

### Handle Conflicts

If ChangeDiffs have conflicting fields, sync/merge returns **409** with conflict details:

```json
{
  "detail": "All conflicts must be acknowledged...",
  "conflicts": [{"object_type": "dcim.device", "object_id": 42,
    "conflicts": ["name", "status"],
    "conflicting_data": {
      "original": {"name": "old"}, "branch": {"name": "branch-val"}, "main": {"name": "main-val"}
    }}]
}
```

To proceed, re-submit with `"acknowledge_conflicts": true`. Branch values win on acknowledged conflicts.

See [references/conflict-resolution.md](references/conflict-resolution.md) for details.

### Revert a Merged Branch

```http
POST /api/plugins/branching/branches/<id>/revert/
Content-Type: application/json

{"commit": true}
```

Only works on `merged` branches. Returns branch to `ready` state. Async job.

### Archive

```http
POST /api/plugins/branching/branches/<id>/archive/
```

Drops the schema, keeps metadata. Terminal state.

## Async Job Polling Pattern

All heavy operations return a Job object with a `url` field. Poll it:

```http
GET <job_url>
```

Job `status` values: `pending`, `running`, `completed`, `errored`, `failed`. Wait for a terminal status. Use exponential backoff (start 1s, max 30s).

**Permissions required:** `netbox_branching.sync_branch`, `netbox_branching.merge_branch`, `netbox_branching.revert_branch`, `netbox_branching.archive_branch`, and (v1.0+) `netbox_branching.migrate_branch` for applying pending migrations.

## Branch-Aware Models

Most DCIM, IPAM, Circuits, Tenancy, Virtualization, VPN, and Wireless models support branching. **NOT branched** (global/immediate): custom fields, webhooks, event rules, export templates, saved filters, all `core.*` models.

> **Discovery endpoint:** `GET /api/plugins/branching/branchable-models/` lists exactly which models are branched on your install — use it instead of guessing. (It was missing on some older 0.8.x builds but is present across the supported 4.4.1+ / plugin 1.0.x range.) In practice, most core operational models (DCIM, IPAM, Circuits, etc.) are branched; infrastructure models (custom fields, webhooks, core.*) and plugin models are exempt.

See [references/branch-aware-models.md](references/branch-aware-models.md).

## Integration with Change Requests

NetBox Branching works with the **netbox-changes** plugin for governed change workflows:

1. Create a Change Request (CR) in netbox-changes
2. Create a branch for that CR
3. Make changes in branch context
4. Submit CR for review/approval
5. Merge branch after CR approval

When integrated, merges are blocked if the associated CR isn't approved.

See the [netbox-changes skill](../netbox-changes/SKILL.md) for CR lifecycle details.

## Anti-Patterns

1. **Always poll after creation** — branch isn't usable until `ready`. Using it in `provisioning` returns 400.
2. **Stale branches** — if `CHANGELOG_RETENTION` is configured and a branch hasn't synced within that window, it becomes stale and cannot sync. Check the `is_stale` / `stale_warning` fields.
3. **Max branch limits** — plugin config sets `max_branches` (total non-archived) and `max_working_branches`. Creation fails with ValidationError if exceeded.
4. **Cannot delete the active branch** — deactivate first.
5. **Merge is all-or-nothing** — any validation failure rolls back the entire transaction.
6. **Conflicts require acknowledgment** — 409 until you pass `acknowledge_conflicts: true`. Branch values win.
7. **Sync cascade deletes** — if a parent object was deleted in main, syncing creates synthetic DELETE records for branch-only children.
8. **Schema = disk space** — each branch copies all branchable tables. Plan capacity for large databases.
9. **`pending-migrations`** — after NetBox upgrades, existing branches need migration before use.
10. **DB privileges** — the PostgreSQL user needs `CREATE ON DATABASE` for schema creation.

## API Endpoints Summary

| Endpoint | Purpose |
|----------|---------|
| `POST /api/plugins/branching/branches/` | Create branch |
| `GET /api/plugins/branching/branches/<id>/` | Branch detail/status |
| `POST .../branches/<id>/sync/` | Sync from main |
| `POST .../branches/<id>/merge/` | Merge to main |
| `POST .../branches/<id>/revert/` | Revert merged branch |
| `POST .../branches/<id>/archive/` | Archive branch |
| `GET /api/plugins/branching/changes/` | ChangeDiff records |
| `GET /api/plugins/branching/branch-events/` | Branch event log |
| `GET /api/plugins/branching/branchable-models/` | Discover branchable models |

## References

- [references/branch-lifecycle.md](references/branch-lifecycle.md) — Complete state machine with all 11 states and transitions
- [references/api-patterns.md](references/api-patterns.md) — Full API examples for every operation
- [references/conflict-resolution.md](references/conflict-resolution.md) — Conflict detection, three-way diff, acknowledgment
- [references/branch-aware-models.md](references/branch-aware-models.md) — What's branched, what's exempt, discovery
