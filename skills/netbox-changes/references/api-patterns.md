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
- `policy` — PK of a Policy. **Required (v1.0+)** — a non-null FK on every CR; a POST without `policy` fails. Also drives merge gating.
- `status` — must be `draft` or `needs-review` (initial choices only)
- `priority` — integer 1 (low) to 5 (high)

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
- `status` — one of: `pending`, `comment`, `changes-requested`, `approved`, `rejected`
- The review automatically records which changes it has seen

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
    "description": "Requires 2 approvals from network team"
}
```

Response includes a `rule_count` field.

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
