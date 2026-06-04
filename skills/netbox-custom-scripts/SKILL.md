---
name: netbox-custom-scripts
description: >
  How to write NetBox custom scripts for automation, data validation, and
  bulk operations. Use when building scripts that run inside NetBox — covers
  the Script class, form variables, ORM access, logging, transactions,
  scheduling, and common patterns.
license: Apache-2.0
---

# NetBox Custom Scripts

> **Your knowledge of NetBox custom scripts may be outdated.** The Script class API, variable types, and job framework evolve between NetBox releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Custom scripts docs | `https://netboxlabs.com/docs/netbox/customization/custom-scripts/` | Script class, variables, scheduling |
| NetBox repo | `https://github.com/netbox-community/netbox` | Script framework source code |
| NetBox MCP server | If configured — verify object state before/after script execution | Testing script output |

## FIRST: Verify Environment

Custom scripts run inside NetBox. Confirm your NetBox instance is running and you can access the scripts endpoint:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/extras/scripts/" | python -m json.tool
```

You should see a list of installed scripts. Scripts are loaded from `SCRIPTS_ROOT` (default: `/opt/netbox/netbox/scripts/`).

---

Use this skill when writing or debugging custom scripts that run inside NetBox. For plugin development (new models, views, APIs), see [netbox-plugin-development](../netbox-plugin-development/SKILL.md).

## Quick Reference

### Minimal Script

```python
from extras.scripts import Script, StringVar, ObjectVar
from dcim.models import Site

class HelloWorld(Script):
    class Meta:
        name = "Hello World"
        description = "A minimal example script"
        commit_default = False

    site = ObjectVar(model=Site)

    def run(self, data, commit):
        self.log_success(f"Selected site: {data['site']}", data['site'])
        return "Done"
```

### Log Methods

| Method | Level | Side Effect |
|--------|-------|-------------|
| `self.log_debug(msg, obj)` | DEBUG | — |
| `self.log_info(msg, obj)` | INFO | — |
| `self.log_success(msg, obj)` | SUCCESS | — |
| `self.log_warning(msg, obj)` | WARNING | — |
| `self.log_failure(msg, obj)` | FAILURE | Sets `self.failed = True` |

Both `msg` and `obj` are optional. Markdown is supported in messages. If `obj` has `get_absolute_url()`, it becomes a clickable link in the UI.

### Common Imports

```python
from extras.scripts import Script, AbortScript
from extras.scripts import (
    StringVar, TextVar, IntegerVar, BooleanVar, ChoiceVar,
    MultiChoiceVar, ObjectVar, MultiObjectVar, FileVar,
    IPAddressVar, IPAddressWithMaskVar, IPNetworkVar,
    DateVar, DateTimeVar, DecimalVar,
)
from dcim.models import Device, Site, Interface, Rack, Region
from ipam.models import IPAddress, Prefix, VLAN, VRF
from tenancy.models import Tenant
from circuits.models import Circuit
```

## Script Structure

### Meta Attributes

| Attribute | Default | Purpose |
|-----------|---------|---------|
| `name` | Class name | Display name in UI |
| `description` | `''` | Description shown in UI |
| `commit_default` | `True` | Default state of commit checkbox |
| `scheduling_enabled` | `True` | Allow scheduled/recurring execution |
| `job_timeout` | `RQ_DEFAULT_TIMEOUT` | Max runtime in seconds |
| `field_order` | Declaration order | Tuple of field names |
| `fieldsets` | `None` | Grouped field layout (overrides `field_order`) |

### Fieldsets

Group related fields in the UI:

```python
class Meta:
    fieldsets = (
        ('Source', ('region', 'site')),
        ('Configuration', ('vlan_id', 'description')),
    )
```

### Module-Level Ordering

Control display order when a file has multiple scripts:

```python
script_order = (ScriptA, ScriptB, ScriptC)
```

## Form Variables

All variables support: `label`, `description`, `default`, `required` (default `True`), `widget`.

| Variable | Key Options | Notes |
|----------|-------------|-------|
| `StringVar` | `min_length`, `max_length`, `regex` | Single-line text |
| `TextVar` | — | Multi-line textarea |
| `IntegerVar` | `min_value`, `max_value` | |
| `BooleanVar` | — | Always optional internally |
| `ChoiceVar` | `choices` (list of `(value, label)`) | Blank choice auto-added |
| `MultiChoiceVar` | `choices` | Multiple selection |
| `ObjectVar` | `model`, `query_params`, `null_option`, `selector`, `quick_add` | Dynamic API-backed dropdown. `selector` (4.5+) shows an advanced object-picker; `quick_add` (4.6.2+) adds an inline "create new" button |
| `MultiObjectVar` | Same as `ObjectVar` | Multiple objects |
| `FileVar` | — | File upload (only available during execution) |
| `IPAddressVar` | — | IPv4/IPv6 without mask |
| `IPAddressWithMaskVar` | — | IP with prefix (e.g., `192.168.1.1/24`) |
| `IPNetworkVar` | `min_prefix_length`, `max_prefix_length` | Network prefix |
| `DateVar` / `DateTimeVar` | — | Date/time pickers |
| `DecimalVar` | `min_value`, `max_value`, `max_digits`, `decimal_places` | |

See [references/script-variables.md](references/script-variables.md) for full details.

### Dynamic Filtering with ObjectVar

Reference other form fields using `$` prefix in `query_params`:

```python
region = ObjectVar(model=Region)
site = ObjectVar(model=Site, query_params={'region_id': '$region'})
rack = ObjectVar(model=Rack, query_params={'site_id': '$site'})
```

The UI dynamically filters each dropdown based on the parent selection.

## ORM Access & Change Logging

Scripts have **full Django ORM access**. Three critical rules:

### 1. Always call `full_clean()` before `save()`

```python
device = Device(name='new-device', site=site, role=role, device_type=dtype)
device.full_clean()   # Validates model constraints — skipping risks data corruption
device.save()
```

### 2. Call `snapshot()` before modifying existing objects

```python
device = Device.objects.get(name='existing')
device.snapshot()     # Required for change log to show a diff
device.status = 'active'
device.full_clean()
device.save()
```

### 3. Use `_changelog_message` for context (optional)

```python
device._changelog_message = 'Bulk status update via maintenance script'
device.save()
```

See [references/orm-patterns.md](references/orm-patterns.md) for query optimization and bulk patterns.

## Transaction Handling

**All script execution is wrapped in `transaction.atomic()`.** This means:

- **`commit=True`**: Changes persist if the script completes without error
- **`commit=False`**: All DB changes are rolled back after `run()` completes (dry-run mode)
- **Any exception**: All changes are rolled back regardless of commit setting
- **Scripts cannot partially commit** — it's all or nothing

### Clean Abort

```python
from utilities.exceptions import AbortScript

if critical_error:
    raise AbortScript("Clear error message")
```

`AbortScript` logs the message as a failure and rolls back all changes — no stack trace in the output.

### `log_failure` Does NOT Abort

Calling `self.log_failure()` sets `self.failed = True` (shows warning in UI) but **does not stop execution**. Use `AbortScript` to actually halt.

## Execution Model

Scripts run as background jobs via Django-RQ:

- **UI/API submission** → job queued to Redis → RQ worker executes
- **CLI**: `python manage.py runscript --commit --user admin module.ClassName`
- **Scheduling**: Set `schedule_at` for future execution, `interval` for recurrence
- **Timeout**: Controlled by `Meta.job_timeout` or global `RQ_DEFAULT_TIMEOUT`

> **NetBox 4.6 background-job changes:** completion **notifications are disabled for scripts running in the background** (4.6) — don't rely on a UI notification to signal a scheduled run finished; check job status instead. NetBox **4.6.2** also prevents **duplicate scheduled background jobs** (re-submitting an already-queued recurring job no longer stacks duplicates).

See [references/execution-model.md](references/execution-model.md) for job lifecycle details.

### Job States

`PENDING` → `RUNNING` → `COMPLETED` | `ERRORED` | `FAILED`

Or `SCHEDULED` → `PENDING` → ...

## Validation Reports (Test Methods)

Methods named `test_*` are auto-detected and run as validation checks:

```python
class CablingAudit(Script):
    class Meta:
        name = "Cabling Audit"

    def test_console_connections(self):
        for device in Device.objects.filter(status='active'):
            ports = ConsolePort.objects.filter(device=device)
            for port in ports:
                if port.connected_endpoints:
                    self.log_success(None, port)  # Count success without message
                else:
                    self.log_failure(f"No console connection: {port}", device)
```

If `run()` is not overridden, NetBox executes `pre_run()` → all `test_*` methods → `post_run()`.

> **NetBox 4.0+**: Reports were merged into scripts. Legacy report log signature `(obj, message)` still works but is deprecated — use `(message, obj)`.

## Anti-Patterns

| Mistake | Consequence | Fix |
|---------|------------|-----|
| Skip `full_clean()` | Data integrity violations | Always validate before save |
| Skip `snapshot()` | Change log shows no diff | Call before modifying existing objects |
| Expect `log_failure` to stop | Script continues running | Use `AbortScript` to halt |
| Long-running script | Killed by timeout | Set `Meta.job_timeout` |
| Import `from netbox.dcim.models` | ImportError | Use `from dcim.models import ...` |
| Assume `FileVar` persists | File only available during execution | Process/copy file in `run()` |
| Rely on `self.request.user` in CLI | Gets `NetBoxFakeRequest` | Handle gracefully |
| Module name conflicts | Import collisions | Avoid naming scripts like installed packages |

## Script File Management

- Scripts are Python files in `SCRIPTS_ROOT` (default: `$INSTALL/netbox/scripts/`)
- Upload via UI (Admin > Custom Scripts) or place files directly
- Each file becomes a `ScriptModule`; classes auto-discovered
- Removing a class from a file soft-deletes its DB record if job history exists (`is_executable=False`)
- Storage backend configurable via Django `STORAGES` (supports S3)
- Scripts can also be synced from a remote **DataSource** (git/S3); on NetBox **4.6.2+** remote-source scripts are **validated on sync**, so a malformed script is caught at sync time rather than first execution
