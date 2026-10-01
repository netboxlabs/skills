# Script Execution Model

## How Scripts Run

Scripts execute as background jobs via Django-RQ (Redis Queue):

1. User submits script form (UI or API)
2. A `Job` record is created in the database
3. Job is queued to Redis
4. RQ worker picks it up and executes
5. All execution is wrapped in `transaction.atomic()`
6. On completion, job status is updated. On NetBox **4.6+** whether the job's user is notified follows the run's `notifications` parameter (`always` / `on_failure` / `never`, defaulted by `Meta.notifications_default`) — poll job status rather than relying on a notification. 4.6.2 also prevents duplicate scheduled background jobs. On **4.7+** the job records `execution_time` (`completed - started`).

## Job Lifecycle States

```
SCHEDULED ──→ PENDING ──→ RUNNING ──→ COMPLETED
                                  ├──→ ERRORED (exception)
                                  └──→ FAILED  (explicit failure)
```

- **PENDING**: Queued for immediate execution
- **SCHEDULED**: Waiting for future execution time
- **RUNNING**: Currently executing in an RQ worker
- **COMPLETED**: Finished successfully
- **ERRORED**: Unhandled exception occurred
- **FAILED**: Explicitly failed (e.g., via `AbortScript`)

## Transaction Behavior

| Scenario | DB Changes |
|----------|-----------|
| `commit=True`, no errors | **Committed** |
| `commit=False`, no errors | **Rolled back** (dry run) |
| Any exception (regardless of commit) | **Rolled back** |
| `AbortScript` raised | **Rolled back** + clean error message |

Scripts **cannot partially commit**. It's all or nothing.

### AbortScript vs Exceptions

```python
from utilities.exceptions import AbortScript

# Clean abort — logs message, no stack trace
raise AbortScript("VLAN pool exhausted, cannot proceed")

# Unhandled exception — full stack trace in job log
raise ValueError("something broke")  # Avoid this for expected errors
```

### log_failure Does NOT Abort

```python
self.log_failure("Missing IP address", device)  # Sets self.failed = True
# Script CONTINUES running — use AbortScript to stop
```

## Execution Methods

### UI Execution

Submit via the script's form page. The commit checkbox controls dry-run behavior.

### API Execution

```http
POST /api/extras/scripts/{id}/
Content-Type: application/json
Authorization: Bearer nbt_abc123.xxxxxxxx

{
    "data": {"site": 1, "new_status": "active"},
    "commit": true,
    "schedule_at": "2026-01-15T03:00:00Z",
    "notifications": "on_failure"
}
```

- Identify the script by numeric ID or `module.ClassName` (e.g. `/api/extras/scripts/my_scripts.BulkUpdate/`). Requires the `extras.run_script` permission plus at least one running RQ worker.
- `data` values for `ObjectVar`/`MultiObjectVar` are **object IDs** (`"site": 1`, `"devices": [1, 2]`), never nested objects.
- **NetBox 4.7+**: the token must have **write enabled** — a read-only token gets HTTP 403 (`"This token does not permit write operations (running a script)."`). 4.6.8+ already enforced object permissions on script write operations via REST.
- **NetBox 4.7.1+**: `data` is validated against the script's declared variables before enqueueing — missing required or invalid values return HTTP 400 with errors nested under `data`; undeclared keys are discarded; object IDs are resolved to instances. Earlier releases passed the payload through largely unvalidated, so clients that relied on extra keys reaching `run()` must be fixed.
- `notifications` (4.6+): `always` / `on_failure` / `never`; defaults to `Meta.notifications_default`.
- Scripts declaring a `FileVar` must be run with `multipart/form-data`, sending `data` as a JSON string alongside the file.

### CLI Execution

```bash
# Synchronous execution (immediate=True)
python manage.py runscript --commit --user admin my_scripts.BulkUpdate

# With data
python manage.py runscript --commit --user admin my_scripts.BulkUpdate \
    --data '{"site": 1, "new_status": "active"}'
```

CLI runs synchronously — does not queue to Redis.

## Scheduling

### One-Time Future Execution

Set `schedule_at` when submitting via API:
```json
{"data": {...}, "commit": true, "schedule_at": "2026-01-15T03:00:00Z"}
```

### Recurring Execution

Set `interval` (minutes) for automatic re-scheduling:
```json
{"data": {...}, "commit": true, "interval": 1440}
```

The job auto-schedules its next run on completion. Disable per-script with `Meta.scheduling_enabled = False`.

## Timeout Control

| Setting | Scope | Default |
|---------|-------|---------|
| `Meta.job_timeout` | Per-script | `None` (falls through) |
| `RQ_DEFAULT_TIMEOUT` | Global | 300 seconds |

For long-running scripts (large bulk operations, external API calls):

```python
class HeavyImport(Script):
    class Meta:
        job_timeout = 1800  # 30 minutes
```

## Job Data Storage

Job results are stored in the `Job` model:
- `data['log']` — List of log entries (message, level, object, URL, timestamp)
- `data['output']` — Return value from `run()` (displayed in Output tab)
- `data['tests']` — Test method results (for validation report pattern)
- `execution_time` *(4.7+)* — read-only duration (`completed - started`), exposed on `GET /api/core/jobs/<id>/`; use it to spot scripts drifting toward `job_timeout`

The run `POST` returns the script's detail representation whose `result` key is the newly enqueued job (`id`, `status`, `url`). Poll `GET /api/core/jobs/<id>/` until `status.value` is `completed`, `errored`, or `failed`.
