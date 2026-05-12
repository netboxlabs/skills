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
- On failure, the branch reverts to its previous stable state:
  - `syncing` → `ready`
  - `merging` → `ready`
  - `reverting` → `merged`
  - `migrating` → `ready` or `pending-migrations`
- **Exception:** `provisioning` failure → `failed` (terminal).

## Working States

The set of "working" (non-terminal, non-merged) branches that count toward `max_working_branches`:
`new`, `ready`, `pending-migrations`, plus all transitional states.

## Key Properties

- **`schema_id`**: 8-char random alphanumeric, assigned at creation. Immutable. Used in the `X-NetBox-Branch` header.
- **`last_sync`**: Timestamp of last successful sync.
- **`is_stale`**: Boolean — true if branch hasn't synced within `CHANGELOG_RETENTION` window.
- **`stale_warning`**: Days remaining before branch becomes stale (null if no retention configured).

## Polling Pattern

After any async operation (create, sync, merge, revert):

1. Capture the Job URL from the response (or poll branch status directly for create).
2. Poll with exponential backoff: 1s, 2s, 4s, 8s... max 30s.
3. Check for terminal status: `completed`, `errored`, `failed`.
4. On completion, verify branch status matches expected state (e.g., `ready` after sync, `merged` after merge).
