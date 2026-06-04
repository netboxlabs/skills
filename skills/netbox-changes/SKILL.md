---
name: netbox-changes
description: >
  Change request management for NetBox Branching. Covers the CR lifecycle
  (draft → needs-review → approved → completed), review workflows, policy
  enforcement, protect_main, and merge gating. Use when working with the
  netbox_changes plugin API.
license: Apache-2.0
---

# NetBox Changes — Change Request Management

> **Your knowledge of NetBox Changes may be outdated.** CR lifecycle, policy enforcement, and API behavior evolve between plugin releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Changes docs | `https://netboxlabs.com/docs/developer/plugins-extensions/changes/` | Configuration, models, lifecycle |
| Changes models | `https://netboxlabs.com/docs/developer/plugins-extensions/changes/models/changerequest/` | CR fields and states |
| NetBox MCP server | If configured — query existing change requests | Live CR state |
| NetBox Platform MCP | If configured — full CR lifecycle operations | Create, review, approve CRs |

## FIRST: Verify Connectivity

Confirm the Changes plugin is installed:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" "$NETBOX_URL/api/plugins/changes/change-requests/" | python -m json.tool
```

If you get 404, the `netbox_changes` plugin is not installed. If 403, your token lacks permissions.

---

The **netbox_changes** plugin adds a code-review-style workflow on top of
[NetBox Branching](../netbox-branching/SKILL.md). Every branch can have one
Change Request (CR) that gates merge via policies and reviews.

**Plugin:** `netbox_changes` 1.0.x (latest v1.0.1) · **NetBox:** 4.4–4.6 · **Base URL:** `/api/plugins/changes/`

> **Plugin v1.0+** changed CR lifecycle semantics vs the 0.4.x line: a branch may hold multiple CRs (one *active* at a time), rejected CRs can be reopened/replaced, `policy` is required, and `changes-requested` is a settable status under an unmet policy. These are flagged inline below.

## Quick Reference

### Endpoints

| Path | Purpose |
|------|---------|
| `change-requests/` | CR CRUD and status management |
| `reviews/` | Submit / list reviews |
| `policies/` | Review policies |
| `policy-rules/` | Rules within policies |
| `comments/` | CR comments |
| `comment-replies/` | Threaded replies |

### CR Statuses (exact values)

`draft` · `needs-review` · `changes-requested` · `approved` · `completed` · `rejected`

### Review Statuses

`pending` · `comment` · `changes-requested` · `approved` · `rejected`

## Core Concepts

### One *Active* CR Per Branch

> **Plugin v1.0+**: a branch may have **multiple** change requests over time, but only **one active** at a time (any status except `completed` or `rejected`). Creating a second *active* CR for a branch fails with a `ValidationError`: `"This branch already has an active change request."` Once a CR is `rejected` or `completed`, the slot frees up — you can open a fresh CR (or reopen the rejected one) for the same branch. Branch deletion cascades to CR deletion.
>
> (In 0.4.x the branch↔CR relation was strictly one-to-one and a second CR raised an `IntegrityError` — do not rely on that against v1.0+.)

### Author/Owner Fields Are Read-Only

`ChangeRequest.owner`, `Review.user`, `Comment.author`, `CommentReply.author`
are all **set automatically** to the current user. Never send them in POST
bodies — they'll be ignored.

## CR Lifecycle

### State Machine

```
CREATE ──→ draft ──→ needs-review ←─────────────────┐
                        │    ↑                       │
                        │    │ (new changes or        │
                        │    │  policy rule change)   │
                        ▼    │                       │
                 changes-requested ──────────────────┘
                        │
                        ▼
                    approved
                        │
                  [branch merge]
                        │
                        ▼
                    completed

Any active status ──→ rejected (manual)
```

New CRs can only be created as `draft` or `needs-review`.

### Automatic Status Transitions

Most transitions happen **automatically**:

| Trigger | From → To |
|---------|-----------|
| Review submitted, policy now met | `needs-review`/`changes-requested` → `approved` |
| Review with `changes-requested` | any active → `changes-requested` |
| New changes made in branch | `approved`/`changes-requested` → `needs-review` |
| Branch merged | `approved` → `completed` (requires merge to complete successfully with actual changes) |
| Policy rule changed or deleted, policy no longer met | `approved` → `needs-review` |

**Manual transitions:** Users can set `draft`, `needs-review`, `changes-requested`,
or `rejected` directly while the policy is unmet (the valid manual set is
`draft` / `needs-review` / `changes-requested` / `rejected`). You **cannot**
manually set `approved` — it's only reachable when the policy passes.
`completed` is only set automatically on merge. A reviewer submitting a
"changes-requested" review also moves the CR to `changes-requested`.

> **Plugin v1.0+**: `changes-requested` is now a valid *manual* status under an unmet policy (in 0.4.x it was auto-only). A `rejected` CR is no longer a dead end — it can be reopened (set back to `draft`/`needs-review`) or replaced by a new CR on the same branch.

See [references/cr-lifecycle.md](references/cr-lifecycle.md) for the complete
transition table.

## Creating a Change Request

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

- `owner` is set automatically — do NOT include it
- `branch` is the branch PK (must exist, must not already have an *active* CR)
- `policy` is **required** (v1.0+) — a non-null FK on every CR; a POST without `policy` fails. (It also drives merge gating.)
- `priority` is an integer (1=low, 5=high)

## Review Workflow

### Submitting a Review

```http
POST /api/plugins/changes/reviews/
{
    "change_request": 1,
    "status": "approved",
    "comments": "LGTM"
}
```

The `user` field is set automatically to the current user. Do not send it.

### Stale Review Tracking

Reviews track which changes they've seen. If new changes are made after a
review, that review becomes **stale** and may need to be re-submitted.

- Stale reviews don't count toward policy evaluation
- Only the **latest non-stale review per user** is considered
- Making changes after approval → CR reverts to `needs-review`, reviewers must re-approve

### Review Iteration Pattern

1. Reviewer submits `changes-requested` → CR status becomes `changes-requested`
2. Author makes fixes in branch → CR reverts to `needs-review` (stale approvals invalidated)
3. Reviewer re-reviews and submits `approved` → if policy met, CR becomes `approved`

## Policy System

A **Policy** contains one or more **PolicyRules**. ALL enabled rules must pass
for the policy to be satisfied.

### PolicyRule Fields

- `min_reviews` — minimum approved (non-stale) reviews required (1–10)
- `reviewers` — M2M to specific Users
- `reviewer_groups` — M2M to Groups
- Only reviews from users in `reviewers` ∪ `reviewer_groups` members count
- `enabled` — disabled rules are skipped

### Critical Gotcha: Zero Rules = Always Fails

A Policy with **zero enabled rules** always returns False. You must create at
least one enabled PolicyRule for a CR to ever reach `approved`.

See [references/policy-system.md](references/policy-system.md) for setup
examples and evaluation details.

## Merge Gating

Merge gating is always active when the plugin is installed — there's no
configuration toggle. Every branch merge requires an approved CR.

If you try to merge a branch without an approved CR, the merge is blocked with:
`"Merging this branch is not permitted."`

## protect_main

**Disabled by default.** When enabled, direct writes to branch-aware models
are blocked unless the user has the bypass permission. All changes must go
through a branch.

```python
# netbox config
PLUGINS_CONFIG = {
    'netbox_changes': {
        'protect_main': True,
    }
}
```

- **Bypass permission (v1.0+):** grant the **`bypass`** custom action on the **Policy** object (NetBox Change Management → Policy in the ObjectPermission form). Superusers get it implicitly. Describe it as "the bypass action on Policy" rather than a single flat permission string — internally the codename is `bypass` while the enforcement check still references `bypass_policy`.
- During branch merge/revert, protect_main is temporarily suspended
- Error when blocked: `"Changes directly to main are not permitted."`

See [references/protect-main.md](references/protect-main.md) for details.

## Complete Workflow

1. **Create branch** (via [netbox-branching](../netbox-branching/SKILL.md) API)
2. **Wait for branch READY** status
3. **Create CR** linked to the branch with a policy
4. **Make changes** in branch context (`X-NetBox-Branch` header)
5. **Set CR** to `needs-review` (or create it as `needs-review`)
6. **Reviewers submit reviews** with `status: approved`
7. **Policy satisfied** → CR auto-transitions to `approved`
8. **Merge branch** → merge gating confirms approved CR → merge proceeds
9. **CR auto-transitions** to `completed`

With `protect_main=True`, step 4 is enforced — no changes without a branch.

## Common Filters

```
GET /api/plugins/changes/change-requests/?status=needs-review
GET /api/plugins/changes/change-requests/?owner=admin
GET /api/plugins/changes/change-requests/?branch=feature-branch
GET /api/plugins/changes/reviews/?change_request_id=1&status=approved
GET /api/plugins/changes/policy-rules/?policy_id=1
```

## Anti-Patterns

1. **Status names** use hyphens: `needs-review`, `changes-requested` (not underscores)
2. **Owner/user/author** fields — read-only, set automatically. Don't POST them.
3. **Zero-rule policy** — always fails. Add at least one enabled rule.
4. **One *active* CR per branch** (v1.0+) — a second *active* CR fails with a ValidationError; a rejected/completed CR frees the slot.
5. **Merge gating is mandatory** — no toggle, always active when plugin installed
6. **protect_main is OFF by default** — must explicitly enable in config
7. **`approved` is not manually settable** — only reached via policy satisfaction

## References

- [references/cr-lifecycle.md](references/cr-lifecycle.md) — Complete state machine with all transitions
- [references/api-patterns.md](references/api-patterns.md) — Full API examples for all endpoints
- [references/policy-system.md](references/policy-system.md) — Policy and rule configuration
- [references/protect-main.md](references/protect-main.md) — protect_main configuration and behavior
