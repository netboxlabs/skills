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

**Plugin:** `netbox_changes` 1.1.x (latest **v1.1.3**, 2026-09-09) · **NetBox:** **4.5.2–4.7** (1.0.2 raised the floor from 4.4 to 4.5.2; 1.1.3 added 4.7) · **Base URL:** `/api/plugins/changes/`

**Pairing with branching:** Changes requires netbox-branching at the line matching your NetBox — **branching 1.1.x on NetBox 4.5.2–4.6**, **branching 1.2.x on NetBox 4.7**. A mismatch prevents NetBox from starting. See [netbox-branching](../netbox-branching/SKILL.md).

> **Plugin v1.0+** changed CR lifecycle semantics vs the 0.4.x line: a branch may hold multiple CRs (one *active* at a time), rejected CRs can be reopened/replaced, `policy` is required, and `changes-requested` is a settable status under an unmet policy. **v1.1.0** added per-policy `is_default` and `require_independent_review`. Flagged inline below; see [Version Notes](#version-notes).

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
| `change-requests/<id>/retrigger/` (POST) | Re-emit event rules for a CR (`event_retriggered`); needs `change` permission and a write-enabled token |

### CR Statuses (exact values)

`draft` · `needs-review` · `changes-requested` · `approved` · `completed` · `rejected`

### Review Statuses

`pending` · `comment` · `changes-requested` · `approved` · `rejected`

`pending` is a "review requested, not yet submitted" placeholder — never POST it. The review form hides it (1.1.1+) and the REST API rejects it: `pending is not a valid status for a submitted review.`

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
- *(1.1.0+)* the policy flagged `is_default: true` (at most one) is **pre-selected in the UI form only** — the REST API does not fill it in. Find it with `GET /api/plugins/changes/policies/?is_default=true` and pass its `id`.
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

### Policy Fields *(1.1.0+)*

- `is_default` — pre-selected on the UI CR form; at most one policy (validation error `A default policy already exists: <name>.`). No effect on the REST API.
- `require_independent_review` — the CR **owner's own reviews never count** toward this policy's rules. Owner can still post reviews (they're recorded, just excluded). If the owner is a rule's only eligible reviewer, the rule is unsatisfiable for their CRs. Default `false` (owner reviews count, as in 1.0.x).

### PolicyRule Fields

- `min_reviews` — minimum approved (non-stale) reviews required, **0–10**. `0` makes the rule always pass (not recommended).
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

If you try to merge a branch without an approved CR, the UI reports
`No change request has been approved for this branch.` Via REST, the branching
`merge/` endpoint still returns 200 + Job; the **job** then errors with
`Merging this branch is not permitted.` — poll the job, don't trust the 200.
Dry-run merges (`"commit": false`) bypass the gate.

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
- *(NetBox 4.7)* `?background=true` bulk writes run in a worker with **no active branch** even when sent with `X-NetBox-Branch` (the header isn't carried to the job), so protect_main rejects them with the error above — inspect the job, the request itself returns 202. Use synchronous bulk writes in branch context.

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
8. **Default policy is UI-only** *(1.1.0+)* — the API never auto-fills `policy`; look up `?is_default=true` and pass it
9. **Self-approval under `require_independent_review`** *(1.1.0+)* — the owner's `approved` review is recorded but ignored; a rule whose only eligible reviewer is the owner can never pass for them
10. **Never POST review status `pending`** — rejected by the API
11. **You can only edit/delete your own reviews, comments, and replies** *(1.0.2+)* — including via bulk PATCH/DELETE (403 `You can only modify your own objects.`); superusers get no exemption
12. **Malformed bulk request bodies** *(1.1.3+)* — a bulk PATCH/DELETE body that isn't a valid list of `{"id": ...}` objects is now passed through to NetBox's own bulk handler on every supported NetBox version, so the error is NetBox's (on 4.7: per-object `{"detail", "errors": [...]}`), not a plugin message. Fix clients that matched on the old plugin error text

## Version Notes

### Plugin 1.1.3 (2026-09-09) — NetBox 4.5.2–4.7
- NetBox 4.7 support. Malformed bulk requests no longer pre-validated by the plugin (see anti-pattern 12). Requires branching 1.2.x on 4.7, 1.1.x on ≤4.6.

### Plugin 1.1.0–1.1.2 (2026-06 → 2026-07) — NetBox 4.5.2–4.6
- **1.1.0:** `Policy.is_default` (UI pre-select), `Policy.require_independent_review`, Policies/Policy Rules nav hidden without `view_policy`/`view_policyrule`, change summary shown on the CR detail page, status/priority help tooltips.
- **1.1.1:** review form can't submit `pending`; NetBox 4.5.x migration compatibility fix.
- **1.1.2:** migration marked `fake_on_branch` so it applies cleanly to branch schemas.

### Plugin 1.0.x (2026-05 → 2026-06)
- **1.0.0:** multiple CRs per branch (one active), reopen rejected CRs, `bypass` permission, owner notifications, `retrigger/` action, `context.last_branch_change` in CR webhook payloads.
- **1.0.2:** **minimum NetBox 4.5.2**; edit/delete of reviews, comments, replies restricted to their owner (also in bulk); `change_comment` needed to resolve/unresolve.

### NetBox 4.7 (2026-09-02) — what changes for CR automation
- Event-rule conditions gain `changed`/`unchanged` and `snapshots.prechange.*` paths: fire only when a CR *becomes* approved instead of on every save — see [references/api-patterns.md](references/api-patterns.md#event-rules-for-cr-automation).
- Webhook context keys `username` and `request_id` are gone; use `request.user` / `request.id` in body templates.
- Per-object bulk errors and `?background=true` (don't use it in a branch — see protect_main above).

## References

- [references/cr-lifecycle.md](references/cr-lifecycle.md) — Complete state machine with all transitions
- [references/api-patterns.md](references/api-patterns.md) — Full API examples for all endpoints
- [references/policy-system.md](references/policy-system.md) — Policy and rule configuration
- [references/protect-main.md](references/protect-main.md) — protect_main configuration and behavior
