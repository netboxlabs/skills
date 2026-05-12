# Policy System

## Overview

A **Policy** is a container for **PolicyRules**. A CR references one Policy.
For the CR to reach `approved`, ALL enabled rules in the policy must pass.

## How Policy Evaluation Works

1. The policy evaluates all enabled rules against the CR
2. Each rule checks the latest non-stale review per eligible user
3. Reviews are stale if new changes have been made in the branch since the review was submitted
4. The count of `approved` (non-stale) reviews must be ≥ `min_reviews`
5. ALL rules must pass → policy passes → CR can become `approved`

## PolicyRule Fields

| Field | Type | Description |
|-------|------|-------------|
| `policy` | FK → Policy | Parent policy |
| `name` | string | Rule name |
| `description` | string | Optional description |
| `enabled` | bool (default True) | Disabled rules are skipped |
| `min_reviews` | int (1–10) | Minimum approved non-stale reviews required |
| `reviewers` | M2M → User | Specific eligible reviewers |
| `reviewer_groups` | M2M → Group | Eligible reviewer groups |

### Eligible Reviewers

Eligible reviewers = union of direct `reviewers` + all members of `reviewer_groups`.
Only reviews from these users count toward `min_reviews`.

## Critical Gotcha: Zero Rules = Always Fails

A Policy with zero enabled rules **can never be satisfied**. This means:
- CRs with this policy can never reach `approved`
- Branches with these CRs can never merge (merge gating)
- Disabling all rules has the same effect

Similarly, a rule with **zero reviewers** (empty `reviewers` and `reviewer_groups`) effectively always fails — no user's review can count toward `min_reviews`.

**Always ensure at least one enabled PolicyRule exists with at least one eligible reviewer.**

## Setup Example

### 1. Create a Policy

```http
POST /api/plugins/changes/policies/
{
    "name": "Standard Review",
    "description": "Requires approval from network team"
}
```

### 2. Add a Rule

```http
POST /api/plugins/changes/policy-rules/
{
    "policy": 1,
    "name": "Two network engineers",
    "min_reviews": 2,
    "reviewer_groups": [3]
}
```

### 3. Multiple Rules (AND logic)

```http
POST /api/plugins/changes/policy-rules/
{
    "policy": 1,
    "name": "Security team sign-off",
    "min_reviews": 1,
    "reviewer_groups": [5]
}
```

Now the policy requires **both**: 2 approvals from network team AND 1 from security.

## Re-evaluation Triggers

Policy compliance is automatically re-evaluated when:

- **Review submitted** → checks if policy now passes → may auto-approve CR
- **Policy rule saved or deleted** → re-evaluates all CRs under affected policy
- **Policy rule reviewers changed** (reviewers or reviewer_groups modified) → same re-evaluation

If a previously-approved CR's policy is no longer met (e.g., rule tightened),
the CR reverts to `needs-review`.

## Stale Review Mechanics

Reviews track which changes they've seen. When new changes are made in the
branch after a review was submitted:

1. The review becomes stale (`is_stale = True`)
2. Stale reviews are excluded from policy evaluation
3. The CR reverts to `needs-review`
4. Reviewers must submit new reviews
