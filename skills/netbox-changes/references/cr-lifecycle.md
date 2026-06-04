# CR Lifecycle — Complete State Machine

## Status Values

| Status | Label | Color | Description |
|--------|-------|-------|-------------|
| `draft` | Draft | gray | Initial state, not yet submitted for review |
| `needs-review` | Needs review | orange | Awaiting reviewer action |
| `changes-requested` | Changes requested | yellow | Reviewer asked for changes |
| `approved` | Approved | green | Policy satisfied, ready to merge |
| `completed` | Completed | blue | Branch merged, CR finished |
| `rejected` | Rejected | red | CR rejected |

## Initial States

New CRs can only be created with status `draft` or `needs-review`.

## Transition Table

### Automatic Transitions

| From | To | Trigger |
|------|----|---------|
| `needs-review` | `approved` | Review submitted, all policy rules pass |
| `changes-requested` | `approved` | Review submitted, all policy rules pass |
| any active | `changes-requested` | Review with `changes-requested` submitted |
| `approved` | `needs-review` | New changes made in the branch |
| `changes-requested` | `needs-review` | New changes made in the branch |
| `approved` | `needs-review` | Policy rule saved/deleted/modified, policy no longer met |
| `approved` | `completed` | Branch merged (requires merge to complete successfully with actual changes) |

### Manual Transitions (User-Initiated)

Users can set these statuses directly via API/UI:

| To | When Allowed |
|----|-------------|
| `draft` | Always (if policy unmet) |
| `needs-review` | Always (if policy unmet) |
| `changes-requested` | Always (if policy unmet) — see v1.0+ note below |
| `rejected` | Always (if policy unmet) |
| `approved` | **Only when policy is satisfied** — enforced by status validation |
| `completed` | **Never manually** — only set automatically on merge |

When policy is NOT met, valid choices are restricted to:
`draft`, `needs-review`, `changes-requested`, `rejected`.

> **Plugin v1.0+**: `changes-requested` is now directly settable under an unmet
> policy (in 0.4.x it was auto-only, reachable solely when a reviewer submitted a
> "changes-requested" review). A reviewer's "changes-requested" review still moves
> the CR to that status automatically. A `rejected` CR is also no longer terminal —
> it can be reopened by setting it back to `draft`/`needs-review`, or replaced by a
> new CR on the same branch.

## State Diagram

```
                        ┌──────────────────────────────────────┐
                        │          (new changes in branch      │
                        │           or policy rule changed)    │
                        │                                      │
  CREATE ──→ draft ──→ needs-review ←──────────────────────────┤
               ↑          │    ↑                               │
               │          │    │ (review: changes-requested    │
               │          │    │  then new changes pushed)     │
               │          ▼    │                               │
               │   changes-requested ──────────────────────────┘
               │          │         (review: approved,
               │          │          policy met)
               │          ▼
               │      approved ────────────────────────────────┐
               │          │                                    │
               │          │ [branch merge]        (new changes │
               │          ▼                        → needs-review)
               │      completed
               │
               └──── rejected ←── (any active status, manual)
```

## Re-evaluation Triggers

Policy compliance is automatically re-evaluated when:

1. **Review submitted** — checks if policy is now met
2. **Policy rule saved or deleted** — re-evaluates all CRs under the affected policy
3. **Policy rule reviewers changed** (reviewers or reviewer_groups modified) — same re-evaluation
4. **New changes in branch** — reverts approved/changes-requested CRs to needs-review

## Key Behaviors

- **Approval invalidation**: Any new change in a branch after approval reverts the CR to `needs-review`. This ensures reviewers see the latest state.
- **Policy rule changes**: If a policy rule is tightened after approval (e.g., `min_reviews` increased), the CR reverts to `needs-review` if the policy is no longer met.
- **Cascading from reviews**: A single `changes-requested` review immediately sets the CR status, regardless of other approvals.
