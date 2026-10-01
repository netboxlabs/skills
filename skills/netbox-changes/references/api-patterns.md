# API Patterns

Base URL: `/api/plugins/changes/`

## Change Requests

### Create

```http
POST /api/plugins/changes/change-requests/
{
    "name": "Add new switches",
    "branch": 123,
    "policy": 1,
    "status": "needs-review",
    "priority": 3,
    "summary": "Adding new switches to DC1"
}
```

- `owner` is set automatically to the current user — do NOT include
- `branch` — PK of an existing branch (must not already have an *active* CR; v1.0+)
- `policy` — PK of a Policy. **Required (v1.0+)** — a non-null FK on every CR; a POST without `policy` fails. Also drives merge gating. *(1.1.0+)* the UI pre-selects the policy with `is_default: true`; the API does not — resolve it yourself: `GET /api/plugins/changes/policies/?is_default=true`.
- `status` — must be `draft` or `needs-review` (initial choices only)
- `priority` — integer 1 (low) to 5 (high)

Bulk `PATCH`/`DELETE` on this and the other endpoints follow NetBox conventions (list of `{"id": ...}`). *(1.1.3+)* a malformed list is handed straight to NetBox's bulk handler, so the error body is NetBox's — on 4.7 `{"detail": ..., "errors": [{"index": N, "errors": {...}}]}` — rather than a plugin-specific message.

### Update

```http
PATCH /api/plugins/changes/change-requests/1/
{
    "status": "needs-review",
    "summary": "Updated summary"
}
```

Status changes are validated. You cannot set `approved` unless the policy
passes, and cannot set `completed` manually.

### List / Filter

```http
GET /api/plugins/changes/change-requests/
GET /api/plugins/changes/change-requests/?status=needs-review
GET /api/plugins/changes/change-requests/?status=draft&status=needs-review
GET /api/plugins/changes/change-requests/?owner=admin
GET /api/plugins/changes/change-requests/?policy=my-policy
GET /api/plugins/changes/change-requests/?branch=feature-branch
```

Response includes a `comment_count` field.

### Delete

```http
DELETE /api/plugins/changes/change-requests/1/
```

## Reviews

### Submit

```http
POST /api/plugins/changes/reviews/
{
    "change_request": 1,
    "status": "approved",
    "comments": "LGTM"
}
```

- `user` is set automatically to the current user — do NOT include
- `status` — one of: `comment`, `changes-requested`, `approved`, `rejected`. `pending` exists as a placeholder value but is rejected on submit (`pending is not a valid status for a submitted review.`); the UI form hides it from 1.1.1.
- The review automatically records which changes it has seen
- *(1.1.0+)* if the CR's policy has `require_independent_review`, a review by the CR owner is stored but never counts toward policy rules
- *(1.0.2+)* `PUT`/`PATCH`/`DELETE` on a review — single or bulk — is allowed only for its `user`; others get 403 (`You can only modify your own objects.` on bulk). Same rule for comments and replies (`author`)

### List / Filter

```http
GET /api/plugins/changes/reviews/?change_request_id=1
GET /api/plugins/changes/reviews/?status=approved
GET /api/plugins/changes/reviews/?user=admin
```

### Stale Check

The `is_stale` property is available on each review object. A review is stale
when new changes have been made in the branch after the review was submitted.

## Policies

### Create

```http
POST /api/plugins/changes/policies/
{
    "name": "Standard Review",
    "description": "Requires 2 approvals from network team",
    "is_default": false,
    "require_independent_review": true
}
```

Response includes a `rule_count` field. `is_default` and `require_independent_review` are 1.1.0+ (also filterable: `?is_default=true`, `?require_independent_review=true`). Setting a second policy's `is_default: true` fails validation while another default exists.

### List

```http
GET /api/plugins/changes/policies/
```

## Policy Rules

### Create

```http
POST /api/plugins/changes/policy-rules/
{
    "policy": 1,
    "name": "Network team approval",
    "min_reviews": 2,
    "reviewer_groups": [3],
    "reviewers": [10, 15],
    "enabled": true
}
```

- `min_reviews` — 1 to 10
- `reviewer_groups` — list of Group PKs
- `reviewers` — list of User PKs
- Eligible reviewers = union of `reviewers` + members of `reviewer_groups`

### Update

```http
PATCH /api/plugins/changes/policy-rules/1/
{
    "min_reviews": 3
}
```

Changing a rule triggers re-evaluation of all CRs using that policy.

### Filter

```http
GET /api/plugins/changes/policy-rules/?policy_id=1
```

## Comments

### Create

```http
POST /api/plugins/changes/comments/
{
    "change_request": 1,
    "content": "Should we use a different VLAN here?"
}
```

The `author` field is set automatically to the current user. Comments can
optionally reference a specific diff object. Comments have a nullable
`resolved` timestamp.

### Reply

```http
POST /api/plugins/changes/comment-replies/
{
    "comment": 5,
    "content": "Good point, I'll update it."
}
```

The `author` field is set automatically to the current user.

## Read-Only Fields

All "who created this" fields are set automatically to the current user:

| Model | Field |
|-------|-------|
| ChangeRequest | `owner` |
| Review | `user` |
| Comment | `author` |
| CommentReply | `author` |

**Never include these fields in POST/PUT/PATCH requests.** They are either
ignored or overridden by the server.

## Retrigger Events

```http
POST /api/plugins/changes/change-requests/1/retrigger/
```

Re-emits event rules for the CR when a webhook target missed the original event. Requires `change_changerequest` and a token with write enabled. Emits the `event_retriggered` event type (not the original one) — handle it in your event rules, or treat it as "re-read the CR via REST". The CR webhook payload always carries `context.last_branch_change` (`id`, `time`, `action`, `changed_object_type`, or `null`).

## Event Rules for CR Automation

The plugin's own lifecycle event types (submitted/approved/rejected/completed) drive **in-app notifications only** — an event rule bound to them never fires. Use standard `object updated` events on `netbox_changes.changerequest` with conditions on `status.value`.

**NetBox ≤ 4.6** — fires on *every* save while the CR is approved (re-evaluations, stale-review updates, comment counts), so make consumers idempotent:

```json
{"attr": "status.value", "value": "approved"}
```

**NetBox 4.7+** — fire only on the transition *into* approved using the `changed` operator (snapshot attributes are raw strings, so use `status`, not `status.value`, on the snapshot side):

```json
{
  "and": [
    {"attr": "status.value", "value": "approved"},
    {"attr": "status", "op": "changed"}
  ]
}
```

A condition that references an attribute missing from both snapshots fails closed and logs to `netbox.event_rules`. On 4.7 webhook body templates must use `request.user` / `request.id` — the `username` and `request_id` context keys were removed.
