# Branch API Patterns

Base URL: `https://netbox.example.com`
All examples use `Authorization: Bearer nbt_abc123.xxxxxxxxxxxxxxxx`.

Endpoints and request/response shapes below are identical on plugin 1.1.x (NetBox 4.4.1–4.6) and 1.2.x (NetBox 4.7); `recover/` is 1.2.1+ only.

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
  "display": "add-london-site",
  "name": "add-london-site",
  "status": {"value": "new", "label": "New"},
  "schema_id": "a1b2c3d4",
  "owner": {"id": 1, "username": "admin"},
  "last_sync": null,
  "merged_time": null,
  "merged_by": null,
  "created": "2025-01-15T10:00:00Z"
}
```

Auto-provisions immediately. Poll status until `ready`. The serializer exposes no `is_stale`, `stale_warning`, or `merge_strategy` fields — compute staleness from `last_sync`/`created` vs `CHANGELOG_RETENTION`.

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
  "detail": "All conflicts must be acknowledged before this action can proceed.",
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

**Strategy:** the request body has no strategy field — REST merges always use **iterative**. Squash is selected on the UI merge form only. To squash-merge a branch driven by automation, do the final merge from the UI.

**Validators run in the job, not the endpoint.** With a merge validator failing (e.g. Changes plugin: no approved CR), `merge/` still returns 200 + Job; the job then finishes `errored` with `Merging this branch is not permitted.` Always check the job result. `{"commit": false}` dry runs skip validators.

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

Synchronous — returns the updated branch representation (no Job). Drops the PostgreSQL schema; branch metadata and events retained. Terminal state; cannot be reverted afterwards.

Valid only from `merged` (400 `Only merged branches can be archived.` otherwise). Archive validators (`archive_validators`) can also block it: 400 `Archiving this branch is not permitted.`

## Recover a Stuck Branch *(1.2.1+, NetBox 4.7)*

```http
POST /api/plugins/branching/branches/7/recover/
Content-Type: application/json

{}
```

Synchronous; returns the updated branch. Only for branches in a transitional status (`provisioning`, `syncing`, `migrating`, `merging`, `reverting`) whose job is no longer running (worker killed). Requires `netbox_branching.change_branch`.

| Stuck in | Reset to |
|----------|----------|
| `provisioning` | `failed` — delete and recreate the branch |
| `syncing` | `ready` |
| `migrating` | `pending-migrations` |
| `merging` | `ready` (transaction rolled back; nothing reached main) |
| `reverting` | `merged` |

Options:
- `{"force": true}` — recover even if a job still *appears* to be running. 400 `A job for this branch still appears to be running. Pass force=true to recover it anyway.` without it. Only use when certain the worker is dead.
- `{"retry": true}` — re-run the interrupted operation after reset. Honoured for `syncing` and `migrating` only; ignored for merge/revert (their `commit` flag isn't stored, so a retry could commit a dry run) and provisioning.

400 `Branch is not in a transitional status.` if the branch isn't stuck. The same recovery runs hourly by default (`auto_recover_stuck_branches`), without retry.

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

Returns all models that support branching, with `app_label`, `model`, `verbose_name`, and `verbose_name_plural` fields. Available on both supported lines (1.1.x and 1.2.x). See [branch-aware-models.md](branch-aware-models.md) for the practical heuristics.

## Background Bulk Writes — Do Not Use in a Branch *(NetBox 4.7)*

```http
POST /api/dcim/sites/?background=true      # WRONG inside a branch
X-NetBox-Branch: a1b2c3d4
```

NetBox 4.7's `?background=true` enqueues the bulk write as a job and rebuilds the request in the worker from a snapshot carrying only host/forwarding headers. `X-NetBox-Branch`, cookies, and `?_branch=` are not carried, so the job runs with **no active branch** and writes to **main** — the 202 gives no hint. Verified against NetBox 4.7.2 and branching 1.2.1. Use synchronous bulk requests in branch context; chunk large batches client-side.

## pynetbox

pynetbox ≥ 7.8.0 ships a branching integration on its plugin-extension framework; `nb.activate_branch()` predates it and works on older 7.x too. Docs: <https://pynetbox.readthedocs.io/en/latest/branching/>.

```python
import pynetbox
from pynetbox.extensions import BranchingExtension

nb = pynetbox.api(
    "https://netbox.example.com",
    token="nbt_abc123.xxxxxxxxxxxxxxxx",
    extensions=[BranchingExtension],   # adds branch.sync()/merge()/revert()/archive()
)

branch = nb.plugins.branching.branches.create(name="add-london-site")
# poll until str(branch.status) == "Ready" (re-fetch with .get(branch.id))

with nb.activate_branch(branch):       # sets X-NetBox-Branch: <schema_id> on the session
    nb.dcim.sites.create(name="London", slug="london", status="planned")

try:
    job = branch.merge(commit=True)    # returns a Jobs record; poll job status
except pynetbox.RequestError as exc:
    if exc.req.status_code == 409:     # exc.error holds the conflicts payload
        job = branch.merge(commit=True, acknowledge_conflicts=True)
    else:
        raise
```

`nb.plugins.branching.changes.filter(branch_id=branch.id)` returns `ChangeDiff` rows with `original_data`/`modified_data`/`current_data`/`conflicts` as plain dicts. Without the extension the endpoints still work for plain CRUD; only the action methods and JSON-field handling are missing.

## Required Permissions

| Action | Permission |
|--------|-----------|
| Sync | `netbox_branching.sync_branch` |
| Merge | `netbox_branching.merge_branch` |
| Migrate | `netbox_branching.migrate_branch` |
| Revert | `netbox_branching.revert_branch` |
| Archive | `netbox_branching.archive_branch` |
| Recover *(1.2.1+)* | `netbox_branching.change_branch` |

Standard CRUD on branches uses `netbox_branching.add_branch`, `change_branch`, `delete_branch`, `view_branch`.
