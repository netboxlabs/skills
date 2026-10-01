# Branch Lifecycle — Complete State Machine

## All 11 States

| Status | Label | Terminal? | Can Read/Write? |
|--------|-------|-----------|-----------------|
| `new` | New | No | No |
| `provisioning` | Provisioning | No | No |
| `ready` | Ready | No | **Yes** |
| `syncing` | Syncing | No | No |
| `migrating` | Migrating | No | No |
| `merging` | Merging | No | No |
| `reverting` | Reverting | No | No |
| `merged` | Merged | Semi¹ | Read-only |
| `archived` | Archived | Yes | No (schema dropped) |
| `pending-migrations` | Pending Migrations | No | No |
| `failed` | Failed | Yes | No |

¹ `merged` allows revert and archive actions but no data changes.

## State Transition Diagram

```
  CREATE
    │
    ▼
  [new] ──auto──▶ [provisioning] ──success──▶ [ready]
                       │                        │  ▲
                       │ fail                    │  │
                       ▼                        │  │
                    [failed]                    │  │
                                                │  │
                    ┌───────────────────────────┘  │
                    │                               │
                    ├──sync──▶ [syncing] ──success──┤
                    │                               │
                    ├──migrate▶ [migrating]─success──┤
                    │                               │
                    └──merge──▶ [merging] ──success──▶ [merged]
                                                        │  │
                                              revert────┘  │
                                                │          │
                                           [reverting]     │
                                                │       archive
                                             success       │
                                                │          ▼
                                             [ready]   [archived]

  NetBox upgrade on existing branch:
    [ready] ──upgrade──▶ [pending-migrations] ──migrate──▶ [migrating] ──▶ [ready]
```

## Transition Failure Behavior

- **Transitional states** (`provisioning`, `syncing`, `migrating`, `merging`, `reverting`) cannot be interrupted.
- On a caught failure, the branch reverts to its previous stable state:
  - `syncing` → `ready`
  - `merging` → `ready`
  - `reverting` → `merged`
- **Exceptions:** `provisioning` failure → `failed` (terminal); `migrating` failure (a migration raised) → `failed` — the schema may be inconsistent and must not be activated.

## Stuck Branches and Recovery *(1.2.1+, NetBox 4.7)*

If the RQ worker dies mid-operation (OOM kill, `kill -9`, evicted container), neither the success nor the failure path runs: the branch keeps its transitional status indefinitely, its actions are disabled, and the job still shows as running. Recovery resets the status to what a clean failure would have left, and marks the orphaned job failed:

| Interrupted | Reset to | Data state |
|-------------|----------|------------|
| `provisioning` | `failed` | Partial schema can't be resumed — delete and recreate |
| `syncing` | `ready` | Sync rolled back; sync again |
| `migrating` | `pending-migrations` | Each migration is its own transaction; re-run Migrate to apply the rest |
| `merging` | `ready` | Rolled back; nothing reached main |
| `reverting` | `merged` | Rolled back; changes still in main |

- **Automatic:** hourly job, `auto_recover_stuck_branches: True` (default). Never re-runs the operation.
- **On demand:** **Recover** button on the branch view, or `POST .../branches/<id>/recover/` (`change_branch` permission; `force`/`retry` options — see [api-patterns.md](api-patterns.md#recover-a-stuck-branch-121-netbox-47)).
- A branch is only recovered once its job is demonstrably not running (job record, RQ, or `job_timeout + stuck_job_grace_period` elapsed).
- 1.1.x has no recovery mechanism.

## Working States

The set of "working" (non-terminal, non-merged) branches that count toward `max_working_branches`:
`new`, `ready`, `pending-migrations`, plus all transitional states.

## Key Properties

- **`schema_id`**: 8-char random alphanumeric, assigned at creation. Immutable. Used in the `X-NetBox-Branch` header.
- **`last_sync`**: Timestamp of last successful sync (`null` until first sync; the branch's `created` time then serves as the sync point).
- **`merged_time` / `merged_by`**: Set on merge, cleared on revert.
- **Staleness is not an API field.** A branch is stale — and can no longer sync — once `now − (last_sync or created)` exceeds NetBox's `CHANGELOG_RETENTION` (default 90 days). The UI shows a warning `stale_warning_threshold` days beforehand (default 7). Automation must compute this from `last_sync`/`created`; a sync attempt on a stale branch returns a Job that errors with `Syncing this branch is not permitted.`

## Polling Pattern

After any async operation (create, sync, merge, revert):

1. Capture the Job URL from the response (or poll branch status directly for create).
2. Poll with exponential backoff: 1s, 2s, 4s, 8s... max 30s.
3. Check for terminal status: `completed`, `errored`, `failed`.
4. On completion, verify branch status matches expected state (e.g., `ready` after sync, `merged` after merge).
