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

**Plugin:** `netbox_branching` — two supported lines, chosen by NetBox version:

| NetBox | Plugin line | Latest |
|--------|-------------|--------|
| **4.7.x** | **1.2.x** (`min_version 4.7.0`; will not load on ≤4.6) | 1.2.1 (2026-09-09) |
| **4.4.1–4.6.x** | **1.1.x** (will not load on 4.7) | 1.1.3 (2026-08-26) |

1.0.x/0.9.x are superseded — upgrade to 1.1.x before moving to NetBox 4.7, then to 1.2.x with the NetBox upgrade. Coverage window for this skill: NetBox 4.5–4.7.

> **1.2.0 is the NetBox 4.7 port.** NetBox 4.7 replaced django-mptt with PostgreSQL `ltree` and moved path/denormalized-field maintenance into DB triggers; 1.2.0 replicates those triggers into every branch schema, so nested hierarchies (regions, locations, tenant groups, module bays…) behave identically inside a branch. Endpoints, permission codenames, the `squash` strategy, and the 11-state lifecycle are the same on both lines; the only 1.2-only API surface is `recover/` (1.2.1). See [Version Notes](#version-notes).

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
| `failed` | Provisioning or migration failed; terminal — delete and recreate |

**Transitional states** (`provisioning`, `syncing`, `migrating`, `merging`, `reverting`) cannot be interrupted. On a *caught* failure they revert to the previous stable state (`syncing`/`merging` → `ready`, `reverting` → `merged`), except `provisioning` and `migrating` → `failed`.

> **Killed worker (OOM, `kill -9`, evicted pod):** no failure path runs, so the branch stays in the transitional status and its actions are disabled. *(1.2.1+ / NetBox 4.7)* an hourly job resets it (`auto_recover_stuck_branches`, default `True`), or do it on demand: `POST /api/plugins/branching/branches/<id>/recover/` (needs `change_branch`; `{"force": true}` if a job still *looks* running, `{"retry": true}` to re-run an interrupted sync/migrate). Reset targets: `provisioning`→`failed`, `syncing`/`merging`→`ready`, `migrating`→`pending-migrations`, `reverting`→`merged`. On 1.1.x there is no recovery endpoint.

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
- **Iterative** (default) — replays every branch `ObjectChange` onto main chronologically, preserving per-change user/time attribution. Use unless it fails.
- **Squash** — collapses to one net operation per object with FK-dependency ordering (any-length FK cycles since 1.1.1). CREATE+DELETE = skip. Attribution in main's changelog is the merging user. Use to recover from duplicate-object or intermediate-state failures.

> **REST has no strategy parameter.** `merge/` accepts only `commit` and `acknowledge_conflicts`; API merges run **iterative**. Squash is chosen on the UI merge form only (`merge_strategy` is not in the REST serializer). Merge validators (e.g. the Changes plugin's approved-CR gate) are checked by the *job*, not the endpoint: the request returns 200 + Job, then the job errors with `Merging this branch is not permitted.` Dry runs (`commit: false`) skip validators.

### Handle Conflicts

If ChangeDiffs have conflicting fields, sync/merge returns **409** with conflict details:

```json
{
  "detail": "All conflicts must be acknowledged before this action can proceed.",
  "conflicts": [{"id": 6, "object_type": "dcim.device", "object_id": 42, "object_repr": "switch-01",
    "action": {"value": "update", "label": "Updated"},
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

Only `merged` branches (400 otherwise). Synchronous — returns the updated branch, no job. Drops the schema, keeps metadata and event history; an archived branch can no longer be reverted. Terminal state. *(1.1.2+)* set `auto_archive_days` to archive merged branches automatically after *n* days.

## Async Job Polling Pattern

All heavy operations return a Job object with a `url` field. Poll it:

```http
GET <job_url>
```

Job `status` values: `pending`, `running`, `completed`, `errored`, `failed`. Wait for a terminal status. Use exponential backoff (start 1s, max 30s).

**Permissions required:** `netbox_branching.sync_branch`, `netbox_branching.merge_branch`, `netbox_branching.revert_branch`, `netbox_branching.archive_branch`, `netbox_branching.migrate_branch` (apply pending migrations), and `netbox_branching.change_branch` for `recover/` (1.2.1+).

## Branch-Aware Models

Most DCIM, IPAM, Circuits, Tenancy, Virtualization, VPN, and Wireless models support branching. **NOT branched** (global/immediate): custom fields, webhooks, event rules, export templates, saved filters, all `core.*` models.

> **Discovery endpoint:** `GET /api/plugins/branching/branchable-models/` lists exactly which models are branched on your install — use it instead of guessing. (Present on both supported lines, 1.1.x and 1.2.x.) In practice, most core operational models (DCIM, IPAM, Circuits, etc.) are branched; infrastructure models (custom fields, webhooks, core.*), the branching and Changes plugins' own models, and anything in `exempt_models` are not. Other plugins' change-logged models are branched by default.

See [references/branch-aware-models.md](references/branch-aware-models.md).

## Integration with Change Requests

NetBox Branching works with the **netbox-changes** plugin for governed change workflows:

1. Create a Change Request (CR) in netbox-changes
2. Create a branch for that CR
3. Make changes in branch context
4. Submit CR for review/approval
5. Merge branch after CR approval

When integrated, merges are blocked if the associated CR isn't approved. Via REST the block is not a 4xx: `merge/` returns 200 + Job and the job errors with `Merging this branch is not permitted.` (the UI shows the reason, `No change request has been approved for this branch.`). Dry-run merges are not gated.

**Pairing:** Changes 1.1.3 works with branching 1.1.x on NetBox ≤4.6 and branching 1.2.x on 4.7 — the two plugins must match the running NetBox version or NetBox will not start. See the [netbox-changes skill](../netbox-changes/SKILL.md) for CR lifecycle details.

## Anti-Patterns

1. **Always poll after creation** — branch isn't usable until `ready`. Using it in `provisioning` returns 400.
2. **Stale branches** — a branch whose `last_sync` (or `created`, if never synced) is older than NetBox's `CHANGELOG_RETENTION` (default 90 days) is stale and can no longer sync: the sync job errors with `Syncing this branch is not permitted.` The REST representation has **no** `is_stale`/`stale_warning` fields — compute `now − (last_sync or created)` yourself; the UI warns `stale_warning_threshold` days ahead (default 7). Sync before retention expires or archive/delete.
3. **Max branch limits** — plugin config sets `max_branches` (total non-archived) and `max_working_branches`. Creation fails with ValidationError if exceeded.
4. **Cannot delete the active branch** — deactivate first.
5. **Merge is all-or-nothing** — any validation failure rolls back the entire transaction.
6. **Conflicts require acknowledgment** — 409 until you pass `acknowledge_conflicts: true`. Branch values win.
7. **Sync cascade deletes** — if a parent object was deleted in main, syncing creates synthetic DELETE records for branch-only children.
8. **Schema = disk space** — each branch copies all branchable tables. Plan capacity for large databases.
9. **`pending-migrations`** — after NetBox upgrades, existing branches need migration before use.
10. **DB privileges** — the PostgreSQL user needs `CREATE ON DATABASE` for schema creation (NetBox 4.7 needs the same privilege to auto-install the `ltree` extension).
11. **`?background=true` + `X-NetBox-Branch` writes to main** *(NetBox 4.7)* — see [NetBox 4.7 Interactions](#netbox-47-interactions).
12. **Programmatic writes need `obj.snapshot()`** — scripts, `nbshell`, and plugin jobs that mutate objects in a branch without a pre-change snapshot get empty `prechange_data`: the diff can't render before/after and **conflict detection silently no-ops** for that object. Call `snapshot()` before mutating.

## API Endpoints Summary

| Endpoint | Purpose |
|----------|---------|
| `POST /api/plugins/branching/branches/` | Create branch |
| `GET /api/plugins/branching/branches/<id>/` | Branch detail/status |
| `POST .../branches/<id>/sync/` | Sync from main |
| `POST .../branches/<id>/merge/` | Merge to main |
| `POST .../branches/<id>/revert/` | Revert merged branch |
| `POST .../branches/<id>/archive/` | Archive merged branch (sync, returns branch) |
| `POST .../branches/<id>/recover/` | Reset a branch stuck in a transitional status *(1.2.1+)* |
| `GET /api/plugins/branching/changes/` | ChangeDiff records |
| `GET /api/plugins/branching/branch-events/` | Branch event log |
| `GET /api/plugins/branching/branchable-models/` | Discover branchable models |

## NetBox 4.7 Interactions

Requires branching **1.2.x**. Verified against NetBox 4.7.2 + plugin 1.2.1.

- **`?background=true` bulk writes ignore the branch header.** The worker rebuilds the request from a snapshot that carries only host/forwarding headers — `X-NetBox-Branch`, cookies, and `?_branch=` are dropped — so the deferred bulk write runs with **no branch active** and lands in **main** (or, with Changes `protect_main`, is rejected). Never combine `?background=true` with `X-NetBox-Branch`; send synchronous bulk requests (≤ a few hundred objects per call) in branch context instead.
- **Hierarchies inside branches:** NetBox 4.7 keeps region/site-group/location/tenant-group/module-bay paths in `ltree` columns maintained by DB triggers. 1.2.0 installs those triggers in each branch schema, so moving or renaming a parent in a branch cascades to descendants in the branch only, and merges replay the parent change (paths on main are recomputed by main's triggers). If you restored NetBox 4.7.0 from `pg_dump`, run `rebuild_ltree_paths --check` on main *before* provisioning new branches — branches copy main's tables and triggers as-is.
- **Custom Objects plugin ≥ 0.6** supports branching custom-object *instances* (version-gated on the plugin, not NetBox); 0.5.x rejects instance writes in a branch. See [netbox-custom-objects](../netbox-custom-objects/SKILL.md).
- **Bulk error shape:** a failed synchronous bulk write in branch context returns 4.7's per-object `{"detail", "errors": [{"index": N, ...}]}` form — same as main. See [netbox-api-integration](../netbox-api-integration/SKILL.md).
- **pynetbox ≥ 7.8.0** ships a `BranchingExtension` (`branch.sync()/merge()/revert()/archive()`) alongside the older `nb.activate_branch()` context manager — see [references/api-patterns.md](references/api-patterns.md#pynetbox).

## Configuration Quick Reference

`PLUGINS_CONFIG['netbox_branching']`. Full list: `docs/configuration.md` in the plugin repo.

| Parameter | Default | Since | Notes |
|-----------|---------|-------|-------|
| `max_branches` / `max_working_branches` | `None` | 0.4 / 0.5 | Total non-archived / non-merged-non-archived caps |
| `exempt_models` | `[]` | 0.5 | `'plugin.model'` or `'plugin.*'`; never exempt a model related to a branched one |
| `job_timeout` | `3600` | 0.8.2 | Sync/merge/revert job timeout (s) |
| `stale_warning_threshold` | `7` | 0.9 | Days before stale to warn in UI; `0` disables |
| `provision_workers` | `4` | 1.1.0 | Parallel copy/index workers; lower to 1–2 on shared DBs |
| `auto_archive_days` | `None` | 1.1.2 | Auto-archive merged branches after *n* days (drops schema → non-revertible) |
| `auto_recover_stuck_branches` | `True` | 1.2.1 | Hourly reset of branches orphaned by a dead worker |
| `stuck_job_grace_period` | `300` | 1.2.1 | Seconds past `job_timeout` before a "running" job is presumed dead |
| `*_validators` (`sync`/`merge`/`migrate`/`revert`/`archive`) | `[]` | 0.6 | Import paths of pre-action validators |

Also add `'netbox_branching.events.add_branch_context'` **before** `extras.events.process_event_queue` in NetBox's `EVENTS_PIPELINE` to get `active_branch` (`id`, `name`, `schema_id` or `null`) in event-rule conditions, webhook payloads, and script `data`.

## Version Notes

### Plugin 1.2.x — NetBox 4.7 only (2026-09-02)
- **1.2.0:** ltree + denormalization triggers replicated into branch schemas. Requires NetBox ≥ 4.7.0; run ≥ 4.7.2 (4.7.0/4.7.1 leaked plaintext tokens created via background bulk requests into job results).
- **1.2.1:** stuck-branch recovery — `recover/` endpoint, `auto_recover_stuck_branches`, `stuck_job_grace_period`; `ObjectChange.undo()` fix for restoring deleted objects on revert.

### Plugin 1.1.x — NetBox 4.4.1–4.6.x (2026-06 → 2026-08)
- **1.1.0:** parallel provisioning (`provision_workers`), up to ~4× faster on large DBs.
- **1.1.1:** squash merge breaks FK cycles of any length; iterative merge of MPTT component models (module bays, inventory items) fixed.
- **1.1.2:** `auto_archive_days`.
- **1.1.3:** conflicts are now recorded on `ChangeDiff` during object updates (previously some update conflicts were missed).

### Plugin 1.0.x (superseded)
- 1.0.4: sync-time conflict resolution preserved at merge; plugin resolver hook for non-change-logged models. 1.0.5: unmodified JSON dict keys preserved on merge. All 1.0.x → upgrade to 1.1.3 (≤4.6) or 1.2.1 (4.7).

## References

- [references/branch-lifecycle.md](references/branch-lifecycle.md) — Complete state machine with all 11 states and transitions
- [references/api-patterns.md](references/api-patterns.md) — Full API examples for every operation
- [references/conflict-resolution.md](references/conflict-resolution.md) — Conflict detection, three-way diff, acknowledgment
- [references/branch-aware-models.md](references/branch-aware-models.md) — What's branched, what's exempt, discovery
