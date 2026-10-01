# Event Rules and Webhooks

## Event Rule System

Event rules (introduced in NetBox 3.7, replacing direct webhook-to-model attachment) are the central automation trigger mechanism.

### Configuration

An event rule consists of:

| Field | Description |
|-------|-------------|
| Object types | Which NetBox models trigger the rule (e.g., Device, IP Address) |
| Event types | `object_created`, `object_updated`, `object_deleted`, `job_started`, `job_completed`, `job_failed`, `job_errored` |
| Conditions | Optional JSON filter on the serialized object data (plus the pre/post-change `snapshots` on 4.7+) |
| Action type | `webhook`, `script`, or `notification` — plus any plugin-registered action type *(4.7)* |
| Action object | The specific Webhook, Script, or NotificationGroup to invoke. Optional on 4.7+ (`action_object_type` may be null for plugin actions that need no target object) |
| Action data | Optional JSON object merged into the event payload passed to the action |
| Enabled | Toggle to activate/deactivate without deleting |

Plugins can register custom event types beyond the built-in set. On **4.7+** they can also register custom action types by subclassing `EventRuleAction` — see [netbox-plugin-development](../../netbox-plugin-development/SKILL.md). An event rule whose plugin action is uninstalled is skipped (not deleted) and exposes `action_is_available: false` in the REST API (`?action_is_available=false` to find them).

### Conditions

Conditions use NetBox's JSON ConditionSet format. They evaluate against the serialized object data (the same representation you'd see from the REST API).

**Example — trigger only when device status is "active":**
```json
{
  "and": [
    {
      "attr": "status.value",
      "value": "active"
    }
  ]
}
```

**Example — trigger for devices in a specific site with a specific role:**
```json
{
  "and": [
    {
      "attr": "site.slug",
      "value": "dc1"
    },
    {
      "attr": "role.slug",
      "value": "leaf-switch"
    }
  ]
}
```

If no conditions are set, the rule fires for every matching event.

Operators: `eq` (default), `gt`, `gte`, `lt`, `lte`, `in`, `contains`, and on **4.7+** `regex`, `changed`, `unchanged`. `negate: true` inverts any condition.

### Firing only on a transition (4.7+ vs. 4.5/4.6)

The condition above fires on **every** update to an active device, not only when status *becomes* active.

> **NetBox 4.7+**: conditions can inspect the event's pre/post-change snapshots. `changed`/`unchanged` take no `value` and compare an attribute across both snapshots; `snapshots.prechange.<attr>` / `snapshots.postchange.<attr>` paths work with any standard operator. Snapshots hold **raw** values (`status` is `"active"`, not `{value,label}`) — write `status`, never `status.value`, in snapshot conditions.

```json
{
  "and": [
    {"attr": "status.value", "value": "active"},
    {"attr": "status", "op": "changed"}
  ]
}
```

Equivalent using an explicit prior value:

```json
{
  "and": [
    {"attr": "snapshots.prechange.status", "value": "planned"},
    {"attr": "snapshots.postchange.status", "value": "active"}
  ]
}
```

Rules on 4.7+:
- A condition referencing an attribute that resolves in neither snapshot (typo, wrong type, `.value` on a snapshot) **fails closed** — the rule does not fire and an error is logged to `netbox.event_rules`. Check worker logs when a rule stops firing after an edit.
- On create events `prechange` is `null`; on delete `postchange` is `null`. `changed` is true for every attribute on create/delete; `snapshots.prechange.<attr>` resolves to `null` on create (matches only `"value": null`). Test for absence with `{"attr": "snapshots.prechange", "value": null}`.
- `changed`/`unchanged` only make sense on `object_updated` (and `object_deleted`); don't put them on job event types.
- `regex` uses Python `re.match` (anchored at the start) against a string attribute.

**On 4.5/4.6** there are no snapshot operators. Keep the condition on `status.value` and do the transition check in the webhook body template (see "Using snapshots for change detection" below), or accept the extra fires and dedupe on the receiver.

### Event Processing

1. Changes are detected during request processing
2. Events are queued (with lazy serialization — data only serialized when consumed)
3. Multiple changes to the same object in one request are coalesced to the final state
4. Delete events eagerly serialize data since the object won't exist afterward
5. Queued events are dispatched to RQ workers asynchronously

**Implication:** There is no guaranteed delivery timing. A webhook might fire milliseconds or seconds after the change, depending on worker load.

---

## Webhook Configuration

A webhook defines the HTTP call that an event rule makes.

### Key Fields

| Field | Description |
|-------|-------------|
| URL | Target endpoint (supports Jinja2 templating) |
| HTTP method | GET, POST, PUT, PATCH, DELETE |
| Content type | e.g., `application/json` |
| Body template | Jinja2 template for the request body |
| Additional headers | Jinja2-templated custom headers |
| Secret | For HMAC-SHA512 payload signing |
| SSL verification | Toggle + optional custom CA file |
| Timeout *(4.7)* | Seconds to wait for the receiver (1–3600). Blank → `WEBHOOK_DEFAULT_TIMEOUT` (default 60). Must be **less than** `RQ_DEFAULT_TIMEOUT` (default 300) or the webhook won't save |

> **NetBox 4.7+ timeout constraints**: `WEBHOOK_DEFAULT_TIMEOUT` must itself be below `RQ_DEFAULT_TIMEOUT` — if you lowered `RQ_DEFAULT_TIMEOUT` to 60 or less and did not set `WEBHOOK_DEFAULT_TIMEOUT`, NetBox refuses to start after the upgrade. The timeout applies separately to connect and read, so a slow-but-streaming receiver can still outlive it; the RQ job timeout is the hard ceiling. Timeouts are logged by `netbox.webhooks` and the job is marked failed. On 4.5/4.6 there is no per-webhook timeout — only `RQ_DEFAULT_TIMEOUT` bounds the request.

### Default Payload

When no body template is set, NetBox sends this JSON structure:

```json
{
  "event": "created",
  "timestamp": "2026-03-06T15:11:23.503186+00:00",
  "object_type": "dcim.site",
  "request": {
    "id": "17af32f0-852a-46ca-a7d4-33ecd0c13de6",
    "method": "POST",
    "path": "/api/dcim/sites/",
    "user": "jstretch"
  },
  "data": {
    "...full REST API representation..."
  },
  "snapshots": {
    "prechange": null,
    "postchange": { "...minimal snapshot (raw field values)..." }
  }
}
```

Version differences in the top-level keys:

| Version | `request` object | `username` / `request_id` |
|---------|------------------|---------------------------|
| 4.7+ | present | **removed** — templates referencing them render empty |
| 4.6 | present | present but deprecated |
| 4.5 | absent | present (the only source of user / request UUID) |

`data` is the REST API representation, so 4.7 API changes show up in webhook payloads too: selection / multi-selection custom fields are `{"value": ..., "label": ...}` objects (`data.custom_fields.env.value`, not `data.custom_fields.env`), and devices/VMs always carry `config_context` (larger payloads). `snapshots` keep raw values on every version.

### Jinja2 Template Context

These variables are available in URL, headers, and body templates:

| Variable | Type | Description |
|----------|------|-------------|
| `event` | string | `"created"`, `"updated"`, `"deleted"` |
| `timestamp` | string | ISO 8601 timestamp |
| `object_type` | string | `"app_label.model_name"` |
| `request` *(4.6+)* | dict | Sanitized request: `request.id` (UUID), `request.method`, `request.path`, `request.path_info`, `request.GET`, `request.user` (username string). Absent on job events with no request |
| `username` | string | **Removed in 4.7.** 4.5/4.6 only — use `request.user` |
| `request_id` | string | **Removed in 4.7.** 4.5/4.6 only — use `request.id` |
| `data` | dict | Full REST API serialization of the object, merged with the rule's `action_data` |
| `snapshots` | dict | `prechange` and `postchange` minimal dicts (raw values) |
| `context` | dict | Only if a plugin registered a webhook callback |

**Using snapshots for change detection:**
```jinja2
{% if snapshots.prechange and snapshots.postchange %}
  {% if snapshots.prechange.status != snapshots.postchange.status %}
    Status changed from {{ snapshots.prechange.status }} to {{ snapshots.postchange.status }}
  {% endif %}
{% endif %}
```

### Custom Body Template Example

Send a Slack-formatted message:
```jinja2
{
  "text": "{{ object_type }} {{ event }}: {{ data.name | default(data.display) }} by {{ request.user }}"
}
```

On 4.6+ use `{{ request.user }}`; `{{ username }}` is 4.5-only and renders empty on 4.7. To support 4.5 through 4.7 in one template: `{{ request.user if request is defined else username }}`. On 4.7+ apply `| header_safe` to any object-derived value placed in a header (strips CR/LF); the filter does not exist on 4.5/4.6, so avoid interpolating user-editable fields into headers there.

### Upgrading to 4.7 with webhooks in flight

`send_webhook()` lost its `username` argument in 4.7. Webhook jobs still queued in Redis when the workers restart on 4.7 code fail with `TypeError`. Before upgrading, stop new changes, let the `rqworker` queues drain (System > Background Tasks shows queued jobs), then upgrade — see [netbox-administration](../../netbox-administration/SKILL.md) for the upgrade procedure. Failed jobs can be requeued only after fixing the caller, so a drained queue is the simplest path.

---

## Security

### HMAC Payload Signing

Set a **secret** on the webhook to enable HMAC-SHA512 signing. NetBox sends the signature in the `X-Hook-Signature` header. The receiver should:

1. Read the raw request body
2. Compute HMAC-SHA512 using the shared secret
3. Compare with the `X-Hook-Signature` header value
4. Reject requests that don't match

**Always set a webhook secret** for production webhooks. Without it, anyone who discovers the receiver URL can send fake events.

### Jinja2 Template Security

Webhook URL, headers, and body templates accept Jinja2 code. This means anyone with permission to create/modify webhooks can execute template logic. **Restrict webhook creation permissions to trusted users only.** Template authoring rights are effectively code-execution-grade — the related ExportTemplate/config-template `environment_params` RCE (CVE-2026-29514) was fixed in NetBox **4.6.1**, so run ≥4.6.1 and keep these permissions tightly scoped.

### SSL Verification

Enable SSL verification and provide a custom CA file if using internal certificate authorities. Disabling SSL verification should only be done in development/testing.

---

## Failure Handling

- Failed webhooks (non-2xx responses) are logged under **System > Background Tasks**
- No automatic retry by default — failed jobs can be manually requeued from the admin UI
- RQ retry configuration may provide limited automatic retry (depends on deployment configuration)
- **Design receivers to be idempotent** — use `request.id` (4.6+; `request_id` on 4.5) to deduplicate in case of retries
- **Timeouts (4.7+)** — a receiver slower than the webhook `timeout` fails the job (`netbox.webhooks` log). Raise the per-webhook timeout (keeping it below `RQ_DEFAULT_TIMEOUT`) or make the receiver acknowledge fast and process async
- **Rule silently stopped firing (4.7+)** — a condition attribute that cannot be resolved fails closed; look for errors from the `netbox.event_rules` logger in worker output

### Troubleshooting

1. **Check Background Tasks** in the NetBox UI for failed webhook jobs
2. **Use the built-in test receiver** during development:
   ```bash
   python netbox/manage.py webhook_receiver  # Listens on port 9000
   ```
3. **Verify network connectivity** from the NetBox worker to the webhook endpoint
4. **Check the RQ worker logs** for detailed error messages
