---
name: netbox-validation
description: >
  NetBox Validation plugin — declarative policy-based validation of network infrastructure data.
  Use when working with validation policies, compliance checks, findings, compliance scores,
  pre-change validation, or agentic safety nets.
license: Apache-2.0
---

# NetBox Validation

NetBox Validation continuously validates infrastructure data against declarative policies using three engines — intent (NetBox data), config analysis (rendered configurations), and graph analysis (infrastructure dependencies). It produces per-device results, aggregated findings, and compliance scores.

> **Your knowledge of NetBox Validation may be outdated.** Check names, API fields, policy pack contents, and engine capabilities evolve between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Validation docs | `https://netboxlabs.com/docs/extensions/validation/` | Full reference, engines, check catalog |
| Validation REST API | `https://netboxlabs.com/docs/extensions/validation/rest-api/` | Endpoint details |
| NetBox Platform MCP | If configured — 5 validation tools | Run validation, query compliance, manage policies |
| NetBox MCP server | If configured — read-only access | Verify devices, policies, scores |

## FIRST: Verify Connectivity

Confirm the Validation plugin is installed and your token has access:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" \
  "$NETBOX_URL/api/plugins/validation/policies/" | python -m json.tool
```

You should see a paginated list of policies (or empty results). If you get 404, the Validation plugin is not installed. If 403, your token lacks validation permissions.

---

## Core Concepts

### Object Hierarchy

```
Policy (scope + rules + triggers)
  -> Run (execution of a policy against devices)
    -> Results (per-device, per-check pass/fail)
      -> Findings (aggregated failures, actionable items)
        -> Compliance Scores (per-device, per-policy, 0-100%)
```

### Policy

A **Validation Policy** defines *what* to check and *where*. Scope filters: sites, regions, site groups, device roles, platforms, tags. Contains rules and controls engine toggles, triggers, and cron schedule.

### Rules

Each rule specifies:

| Field | Description |
|-------|-------------|
| `engine` | `intent`, `config`, or `graph` |
| `category` | Check grouping (addressing, redundancy, topology, etc.) |
| `check_name` | Specific check to run (e.g., `no_duplicate_ips`) |
| `severity` | `critical`, `high`, `medium`, `low`, or `info` |
| `parameters` | Check-specific config (thresholds, patterns, expected values) |

Rules can optionally override the policy scope with their own roles and platforms filters.

### Three Engines

| Engine | Checks | Evaluates | Data Source | Key Strength |
|--------|--------|-----------|-------------|--------------|
| Intent | 42 | NetBox data relationships | NetBox ORM | Fast, no external dependencies |
| Config | 35 | Rendered device configurations | Config analysis engine | Structural analysis, reachability |
| Graph | 16 | Infrastructure dependency graph | NetBox ORM | Blast radius, failure domains |

All three can coexist in a single policy. Config and graph engines are available to Premium tier customers.

### Run Statuses

| Status | Meaning |
|--------|---------|
| `pending` | Created, waiting for execution |
| `running` | Checks are executing |
| `passed` | All checks passed |
| `failed` | One or more checks failed |
| `error` | Engine error during execution |

### Result Statuses

Each per-device, per-check result: `pass`, `fail`, `skip`, `warning`, or `error`.

### Finding Lifecycle

```
open -> acknowledged -> resolved
                    \-> suppressed
```

| Status | When to use |
|--------|-------------|
| `open` | New, needs review |
| `acknowledged` | Reviewed, remediation planned |
| `resolved` | Fixed and confirmed by passing run |
| `suppressed` | Intentional exception, risk accepted |

### Compliance Score

```
score = passed / (passed + failed + error) x 100
```

Skipped checks excluded. Scores are per (device, policy, run) and tracked over time.

---

## Quick Reference — Common Workflows

### Create and Execute a Run

Creating a run does NOT execute it. Two-step process:

```bash
# Step 1: Create the run
RUN=$(curl -s -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual"}')

RUN_ID=$(echo $RUN | python -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Step 2: Execute it (synchronous — returns when complete)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/$RUN_ID/execute/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

The execute endpoint is **synchronous** — it returns results when the run completes. Do not poll.

### Install a Policy Pack

```bash
# List available packs
curl "$NETBOX_URL/api/plugins/validation/policy-packs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Install a pack (creates policy + all rules)
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/install/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

### Query Compliance Scores

```bash
# Fleet-wide latest scores
curl "$NETBOX_URL/api/plugins/validation/compliance/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Scores for a specific device, newest first
curl "$NETBOX_URL/api/plugins/validation/compliance/?device_id=10&ordering=-measured_at" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Scores for a specific policy
curl "$NETBOX_URL/api/plugins/validation/compliance/?policy_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

### Filter Findings

```bash
# Open critical findings
curl "$NETBOX_URL/api/plugins/validation/findings/?status=open&severity=critical" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Acknowledge a finding
curl -X PATCH "$NETBOX_URL/api/plugins/validation/findings/42/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "acknowledged"}'
```

### Run Against a Branch

```bash
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual", "branch_id": 5}'
```

The validation engines read rules from main and execute checks in branch context, seeing branch-overlay data.

### Targeted Run (Narrow Scope)

```bash
# Target specific site(s)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "api", "target_sites": [9]}'

# Validate a single device across all matching policies
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/validate-device/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"device": 101}'
```

### YAML Import/Export

```bash
# Export a single policy
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/export/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Import a policy from YAML
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/import/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d @my-policy.yaml
```

---

## Policy Management

### Policy Scoping

Policies scope devices via any combination of:

| Scope Field | Effect |
|-------------|--------|
| Sites | Only devices at these sites |
| Regions | Only devices at sites in these regions |
| Site Groups | Only devices at sites in these groups |
| Device Roles | Only devices with these roles |
| Platforms | Only devices on these platforms |
| Tags | Only devices with these tags |

Scope fields are AND-combined. An empty scope means all active devices.

### Engine Toggles

| Policy Field | Default | Effect |
|-------------|---------|--------|
| `enable_config_engine` | `false` | Enable config analysis checks for this policy |
| `enable_graph_engine` | `false` | Enable graph resilience checks for this policy |

Intent checks always run. Config and graph checks require their engine to be enabled on the policy AND available at the deployment level.

### Policy Packs

Pre-built policy packs provide one-click installation:

- **14 starter packs** — addressing, cabling, data quality, naming, redundancy, security, leaf/spine baselines, config analysis, pre-change, BGP attributes, power/network/full resilience
- **9 compliance frameworks** — CLOS Fabric, TIA-942, NIS2/DORA, NIST 800-53, NERC CIP, PCI-DSS, MANRS, ISO 27001, HIPAA Security Rule (2026)

Installed packs create regular policies and rules. After installation, customize scope, parameters, triggers, and schedule. See [references/policy-packs.md](references/policy-packs.md).

### Clone and Import

```bash
# Clone a policy (copies all rules and scoping)
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/clone/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Import from YAML
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/import/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d @my-policy.yaml
```

---

## Validation Triggers

### Manual

Create a run via API or click "Run Now" in the UI. Two-step: create run, then execute.

### Branch Merge

Set `trigger_on_branch_merge: true` on the policy. Validation runs automatically when a NetBox Branching branch is merged to main. Catches regressions in the merged data.

### Change Request Submit

Set `trigger_on_cr_submit: true` on the policy. Validation runs when a NetBox Changes change request is submitted for review. The run targets the CR's branch, providing pre-merge safety.

### Scheduled

Set a cron expression on the policy's `schedule` field:

| Expression | Meaning |
|-----------|---------|
| `0 2 * * *` | Daily at 2:00 AM |
| `0 */6 * * *` | Every 6 hours |
| `0 0 * * 0` | Weekly on Sunday at midnight |

Scheduled runs target main (no branch context).

### Run Scoping

All runs support optional target narrowing:

| Field | Type | Effect |
|-------|------|--------|
| `target_sites` | List of site IDs | Only devices at these sites |
| `target_devices` | List of device IDs | Only these specific devices |
| `target_tags` | List of tag slugs | Only devices with these tags |

Target fields AND-combine with the policy scope. They can only narrow, never widen.

---

## Compliance and Findings

### Score Calculation

```
score = passed / (passed + failed + error) x 100
```

Skipped checks excluded. Scores enable trend tracking, audit reporting, change request gating, and executive dashboards.

### Score Dimensions

Compliance scores can be queried by:
- **Device** — per-device compliance across policies
- **Policy** — per-policy compliance across devices
- **Time** — trend over 7d/30d/90d via the compliance dashboard

### Finding Aggregation

Findings group related failures into actionable items. For example, if `no_duplicate_ips` fails for 5 devices, one finding groups all 5. Graph engine findings group by failure point (one finding per power feed or shared failure domain).

### Severity Escalation

Graph engine findings automatically escalate severity based on blast radius:

| Unprotected Devices | Escalation |
|---------------------|------------|
| 1--2 | No change |
| 3--5 | medium -> high |
| 6--10 | high -> critical |
| 10+ | Always critical |

### Finding Triage

1. Filter findings by severity (critical first)
2. Review detail: affected devices, expected vs. actual, remediation
3. For graph findings, check blast radius card
4. Acknowledge findings you plan to fix
5. Fix the underlying issue in NetBox
6. Re-run validation to confirm
7. Mark finding as resolved

---

## Integration Patterns

### Pre-Change Validation Pipeline

```
1. Create Branch          (netbox-branching)
2. Make changes           (normal NetBox API with X-NetBox-Branch header)
3. Submit Change Request  (netbox-changes) -> validation auto-runs
4. Review validation results
5. Approve CR, merge branch
6. Post-merge validation auto-runs (optional)
```

Set `trigger_on_cr_submit: true` for pre-merge safety. Set `trigger_on_branch_merge: true` for post-merge verification.

### Branch Context

When running validation against a branch, pass `branch_id` on the run. The engines read rules from main and execute checks in branch context (seeing proposed additions, modifications, deletions).

Config analysis renders configs from both main and branch for differential analysis (lost reachability, broken BGP sessions, routing regressions).

### Agentic Integration via MCP

The NetBox Labs Platform MCP Server provides 5 validation tools:

| Tool | Purpose |
|------|---------|
| `run_validation` | Create + execute a run for a policy, optionally targeting a branch or devices |
| `get_compliance_analytics` | Compliance scores, trends, score-by-dimension |
| `get_findings_analytics` | Findings filtered by severity, status, category, policy |
| `import_validation_policy` | Import policy from YAML |
| `clone_validation_policy` | Clone policy with all rules and scoping |

Agents that prefer direct API access can use the REST endpoints with bearer token authentication.

### Agentic Workflow Pattern

```
Agent creates branch -> makes changes -> runs validation
  Pass -> creates change request for human review
  Fail -> reads findings, self-corrects, re-validates
```

Validation provides the automated safety net ensuring agents operate within the same policy framework as human engineers.

---

## Anti-Patterns

| Mistake | Why It Fails | Do This Instead |
|---------|-------------|-----------------|
| Create a run and assume it executes | Creating a run sets status to `pending` | POST to `/runs/{id}/execute/` after creation |
| Omit `trigger` field when creating runs | `trigger` is required | Set `trigger` to `"manual"`, `"api"`, or `"schedule"` |
| Use branch integer ID for branch validation | `branch_id` on run creation expects the branch's integer ID, but `X-NetBox-Branch` header uses `schema_id` | Use the correct identifier for each context |
| Install config/graph packs without enabling engine | Config/graph rules produce `skip` results | Set `enable_config_engine` / `enable_graph_engine` on the policy |
| Expect target scoping to widen policy scope | Target fields AND-combine with policy scope | Targets can only narrow — devices outside policy scope are excluded |
| Poll for run completion after execute | Execute is synchronous — returns when done | Read the response directly; no polling needed |
| Run graph checks with device targeting expecting savings | Graph engine loads all site devices regardless | Use site-scoped runs for graph-heavy policies |

---

## API Endpoints Summary

| Endpoint | Methods | Purpose |
|----------|---------|---------|
| `/policies/` | GET, POST | List and create policies |
| `/policies/{id}/` | GET, PUT, PATCH, DELETE | Manage a policy |
| `/policies/{id}/clone/` | POST | Clone with all rules and scoping |
| `/policies/{id}/export/` | POST | Export as YAML |
| `/policies/import/` | POST | Import from YAML |
| `/policies/export-all/` | POST | Export all policies |
| `/policies/{id}/run-for-site/` | POST | Targeted run for site(s) |
| `/policies/{id}/run-for-devices/` | POST | Targeted run for device(s) |
| `/rules/` | GET, POST | List and create rules |
| `/rules/{id}/` | GET, PUT, PATCH, DELETE | Manage a rule |
| `/runs/` | GET, POST | List and create runs |
| `/runs/{id}/` | GET | Run status and summary |
| `/runs/{id}/execute/` | POST | Execute a pending run |
| `/runs/validate-device/` | POST | Run all matching policies for a device |
| `/results/` | GET | Results (filter by run, device, status) |
| `/findings/` | GET, POST | Findings (filter by run, severity, status) |
| `/findings/{id}/` | GET, PUT, PATCH | Update finding status |
| `/compliance/` | GET | Compliance scores (filter by device, policy) |
| `/policy-packs/` | GET | List available policy packs |
| `/policy-packs/{slug}/` | GET | Pack details with rule list |
| `/policy-packs/{slug}/install/` | POST | Install a pack |
| `/policy-packs/{slug}/uninstall/` | POST | Uninstall a pack |

All endpoints under `/api/plugins/validation/`.

---

## References

| Reference | When to Load |
|-----------|-------------|
| [references/api-patterns.md](references/api-patterns.md) | Need full API examples for policy/run/result lifecycle |
| [references/check-reference.md](references/check-reference.md) | Need check names, parameters, or engine details |
| [references/policy-packs.md](references/policy-packs.md) | Need pack catalog, install/customize guidance, framework details |
