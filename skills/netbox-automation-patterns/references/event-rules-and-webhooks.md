# Event Rules and Webhooks

## Event Rule System

Event rules (introduced in NetBox 3.7, replacing direct webhook-to-model attachment) are the central automation trigger mechanism.

### Configuration

An event rule consists of:

| Field | Description |
|-------|-------------|
| Object types | Which NetBox models trigger the rule (e.g., Device, IP Address) |
| Event types | `object_created`, `object_updated`, `object_deleted`, `job_started`, `job_completed`, `job_failed`, `job_errored` |
| Conditions | Optional JSON filter on the serialized object data |
| Action type | `webhook`, `script`, or `notification` |
| Action object | The specific Webhook, Script, or NotificationGroup to invoke |
| Enabled | Toggle to activate/deactivate without deleting |

Plugins can register custom event types beyond the built-in set.

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
  "username": "jstretch",
  "request_id": "17af32f0-852a-46ca-a7d4-33ecd0c13de6",
  "data": {
    "...full REST API representation..."
  },
  "snapshots": {
    "prechange": null,
    "postchange": { "...minimal snapshot..." }
  }
}
```

> **NetBox 4.6:** the top-level `username` and `request_id` fields are **deprecated** (removal in **v4.7**) and superseded by the `request` object (`request.id`, `request.method`, `request.path`, `request.user`). Author new templates against `request.*`; the legacy fields still render on 4.6 but will disappear in 4.7. On NetBox **4.5** only the legacy fields exist (`request` is not present), so target the version range you support accordingly.

### Jinja2 Template Context

These variables are available in URL, headers, and body templates:

| Variable | Type | Description |
|----------|------|-------------|
| `event` | string | `"created"`, `"updated"`, `"deleted"` |
| `timestamp` | string | ISO 8601 timestamp |
| `object_type` | string | `"app_label.model_name"` |
| `request` *(4.6)* | dict | Serialized HTTP request: `request.id` (UUID), `request.method`, `request.path`, `request.user` |
| `username` | string | *(deprecated 4.6, removed 4.7)* User who made the change — use `request.user` |
| `request_id` | string | *(deprecated 4.6, removed 4.7)* UUID correlating changes in one request — use `request.id` |
| `data` | dict | Full REST API serialization of the object |
| `snapshots` | dict | `prechange` and `postchange` minimal dicts |

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

On NetBox 4.5 (no `request` object), use `{{ username }}`; on 4.6+ prefer `{{ request.user }}`. To support both in one template: `{{ request.user if request is defined else username }}`.

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

### Troubleshooting

1. **Check Background Tasks** in the NetBox UI for failed webhook jobs
2. **Use the built-in test receiver** during development:
   ```bash
   python netbox/manage.py webhook_receiver  # Listens on port 9000
   ```
3. **Verify network connectivity** from the NetBox worker to the webhook endpoint
4. **Check the RQ worker logs** for detailed error messages
