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

**Current plugin:** **1.14.3** (2026-10-06). NetBox 4.7 support arrived in 1.14.0; the plugin declares NetBox 4.4–4.7.x, and the docs require NetBox 4.5+ (NetBox Cloud). Run ≥ 1.14.0 on NetBox 4.7. See [Version Notes](#version-notes).

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Validation docs | `https://netboxlabs.com/docs/validation/` | Full reference, engines, check catalog, policy packs |
| Validation REST API | `https://netboxlabs.com/docs/validation/api-reference/` | Endpoint details |
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
    -> Results (per-device, per-check pass/fail/warning/error/skip)
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
| `category` | Check grouping — `addressing`, `redundancy`, `standards`, `completeness`, `security_intent`, `topology`, `reachability`, `power`, `infrastructure`, `routing`, `security`, `baseline`, `overlay` |
| `check_name` | Specific check to run (e.g., `no_duplicate_ips`) |
| `severity` | `critical`, `high`, `medium`, `low`, or `info` |
| `parameters` | Check-specific config (thresholds, patterns, expected values) |

Rules can optionally override the policy scope with their own `roles` and `platforms` filters (lists of IDs).

### Three Engines

| Engine | Checks | Evaluates | Data Source | Key Strength |
|--------|--------|-----------|-------------|--------------|
| Intent | 44 | NetBox data relationships | NetBox ORM | Fast, no external dependencies |
| Config | 35 | Rendered device configurations | Config analysis engine (from NetBox Config Templates) | Structural analysis, reachability |
| Graph | 16 | Infrastructure dependency graph | NetBox ORM | Blast radius, failure domains |

All three can coexist in a single policy. Config and graph engines are available to Premium tier customers. The config engine analyzes configurations rendered from NetBox Config Templates — assign templates to devices (via device, platform, or role) before enabling it.

### Run Statuses

| Status | Meaning |
|--------|---------|
| `pending` | Created, waiting for execution |
| `running` | Checks are executing |
| `passed` | All checks passed |
| `failed` | One or more checks failed |
| `error` | Engine error during execution |

Since plugin 1.12.0 each run is linked to its NetBox background job, bounded by a job timeout, and orphaned runs (worker lost mid-run) are reaped rather than left in `running`.

Each run exposes `job_id` (the core background Job). The plugin settings `job_timeout` (default 3600s) and `run_retention_days` (default 180; `<= 0` disables) govern this: completed runs older than the window are pruned by the hourly housekeeping job, which never prunes in-flight runs and always keeps each policy's most recent run.

### Result Statuses

Each per-device, per-check result: `pass`, `fail`, `skip`, `warning`, or `error`. Results carry `expected`, `actual`, `message`, `remediation` and `extra` (engine-specific detail).

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
# Step 1: Create the run — policy, trigger AND status are required (400 without "status")
RUN=$(curl -s -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual", "status": "pending"}')

RUN_ID=$(echo $RUN | python -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Step 2: Execute it (synchronous — returns when complete)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/$RUN_ID/execute/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

The execute endpoint is **synchronous** — it returns the completed run (status, counts, score). Do not poll. Only `pending` (or `error`) runs can be executed; re-executing a finished run returns 400.

### Install a Policy Pack

```bash
# List available packs (a bare JSON array)
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
# Open critical findings from the latest run of each policy
curl "$NETBOX_URL/api/plugins/validation/findings/?latest=true&status=open&severity=critical" \
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
  -d '{"policy": 1, "trigger": "manual", "status": "pending", "branch_id": 5}'
```

The validation engines read rules from main and execute checks in branch context, seeing branch-overlay data.

### Targeted Run (Narrow Scope)

```bash
# Target specific site(s)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "api", "status": "pending", "target_sites": [9]}'

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

| Scope Field | API field | Effect |
|-------------|-----------|--------|
| Sites | `sites` | Only devices at these sites |
| Regions | `regions` | Only devices at sites in these regions |
| Site Groups | `site_groups` | Only devices at sites in these groups |
| Device Roles | `roles` | Only devices with these roles |
| Platforms | `platforms` | Only devices on these platforms |
| Tags | `device_tags` | Only devices with these tags |

Each is a list of NetBox object IDs. Scope fields are AND-combined. An empty scope means all active devices. **Unknown fields are silently ignored**: a body with `scope_sites` creates an *unscoped* policy with no error, so read `sites` / `roles` back from the response after writing.

```bash
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"name": "Leaf Baseline", "sites": [1, 2], "roles": [3], "device_tags": [7]}'
```

Choice fields (`engine`, `category`, `severity` on rules; `trigger`, `status` on runs; `status`, `severity` on results; `severity`, `category`, `status` on findings) read back as `{"value", "label"}` objects; writes take bare strings. The `status`/`trigger` inside a policy's `latest_run` summary are bare strings. Compare `x["status"]["value"]`, not `x["status"]`.

### Engine Toggles

| Policy Field | Default | Effect |
|-------------|---------|--------|
| `enable_config_engine` | `false` | Enable config analysis checks for this policy |
| `enable_graph_engine` | `false` | Enable graph resilience checks for this policy |
| `trigger_on_cr_submit` | `true` | Run when a change request is submitted |
| `trigger_on_branch_merge` | `false` | Run when a branch is merged |

Intent checks always run. Config and graph checks require their engine to be enabled on the policy AND available at the deployment level.

### Policy Packs

Pre-built policy packs provide one-click installation:

- **14 starter packs** (113 rules) — addressing, cabling, data quality, naming, redundancy, security intent, leaf/spine baselines, config analysis, pre-change config, BGP attributes, power/network/full resilience
- **9 compliance frameworks** (177 rules; Premium) — CLOS Fabric, TIA-942, NIS2/DORA, NIST 800-53, NERC CIP, PCI-DSS, MANRS, ISO 27001, HIPAA Security Rule (2026)

Starter packs are listed by default; framework packs appear (and can be installed) only when the plugin setting `enabled_policy_packs` includes `"compliance"` (default `["starter"]`) — installing a disabled category returns 403. `GET /policy-packs/` returns a bare JSON array (no `count`/`results`); each entry carries `slug`, `name`, `description`, `category` (`starter` | `compliance`), `policy_count`, `rule_count`, `engines`, `installed`, `installed_policy_ids`. Installed packs create regular policies and rules. After installation, customize scope, parameters, triggers, and schedule. See [references/policy-packs.md](references/policy-packs.md).

### Delete Rules and Policies

Since plugin 1.13.0, `DELETE /rules/{id}/` and `DELETE /policies/{id}/` preserve historical run results. Results keep `rule_name` and `check_name` snapshots, and runs and compliance scores keep `policy_name`; after a deletion the `rule` / `policy` FK fields are `null`, so read the `*_name` fields rather than assuming the FK is set. Policy-pack uninstall is no longer refused when runs exist — the pack's policies are removed and their run history is retained under the snapshotted policy name. Deleting a **run** (`DELETE /runs/{id}/`) is the one destructive path: it cascades the run's results, findings and compliance scores.

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

| Trigger | How | Notes |
|---|---|---|
| Manual | API create + execute, or "Run Now" in the UI | Two-step: create (`trigger: "manual"`, `status: "pending"`), then execute |
| Branch merge | `trigger_on_branch_merge: true` | Runs when a NetBox Branching branch is merged to main; catches regressions in the merged data |
| Change request submit | `trigger_on_cr_submit: true` (default on) | Runs when a NetBox Changes change request is submitted; targets the CR's branch for pre-merge safety |
| Scheduled | cron in `schedule` — `0 2 * * *` (daily 02:00), `0 */6 * * *` (every 6 hours), `0 0 * * 0` (weekly) | Targets main (no branch context); minimum interval 5 minutes |

Run `trigger` values: `manual`, `api`, `schedule`, `branch`, `merge`, `cr`.

### Run Scoping

All runs support optional target narrowing:

| Field | Type | Effect |
|-------|------|--------|
| `target_sites` | List of site IDs | Only devices at these sites |
| `target_devices` | List of device IDs | Only these specific devices |
| `target_tags` | List of tag IDs | Only devices with these tags |

Target fields AND-combine with the policy scope. They can only narrow, never widen. (The public API reference shows tag slugs for `target_tags`; the endpoint takes NetBox Tag IDs like the other target fields.)

---

## Compliance and Findings

### Score Dimensions

Compliance scores can be queried by **device** (across policies), **policy** (across devices), **site/role** (`?site_id=`, `?role_id=`), and **time** (7d/30d/90d trend via the compliance dashboard and `/compliance/trend/`).

### Finding Aggregation

Findings group related failures into actionable items. For example, if `no_duplicate_ips` fails for 5 devices, one finding groups all 5. Graph engine findings group by failure point (one finding per power feed or shared failure domain). Each finding carries `affected_devices`, `is_new`, `recurrence_count`, `first_seen`, `policy_id`/`policy_name` and `has_blast_radius`.

### Severity Escalation

Graph engine findings automatically escalate severity based on blast radius:

| Unprotected Devices | Escalation |
|---------------------|------------|
| 1--2 | No change |
| 3--5 | medium -> high |
| 6--10 | high -> critical |
| 10+ | Always critical |

### Finding Triage

1. Filter findings by severity (critical first); use `?latest=true` for current findings only
2. Review detail: affected devices, `expected` vs `actual`, `remediation`
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
| Omit `trigger` or `status` when creating runs | Both are required — 400 `{"status": ["This field is required."]}` (the public API reference omits `status`; the endpoint still requires it) | Body is always `{"policy": <id>, "trigger": "manual", "status": "pending"}` (`trigger` may be `"api"` or `"schedule"`) |
| Scope a policy with `scope_sites` / `scope_roles` / `scope_tags` | Unknown fields are silently dropped — the policy is created **unscoped** (all devices) | Use `sites`, `regions`, `site_groups`, `roles`, `platforms`, `device_tags` (ID lists) and read `sites`/`roles` back from the response |
| Compare a choice field to a bare string | `status`, `trigger`, `engine`, `category`, `severity` read back as `{"value", "label"}` | Compare `["value"]`; write bare strings |
| Use branch integer ID for branch validation | `branch_id` on run creation expects the branch's integer ID, but `X-NetBox-Branch` header uses `schema_id` | Use the correct identifier for each context |
| Install config/graph packs without enabling engine | Config/graph rules produce `skip` results | Set `enable_config_engine` / `enable_graph_engine` on the policy |
| Install a compliance pack when only starter packs are enabled | `POST .../install/` returns 403 | Add `"compliance"` to the plugin setting `enabled_policy_packs` |
| Treat `skip` as a failure | Skips are excluded from scores and never create findings | Read the result `message` — no Config Template, no power ports, engine unavailable |
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
| `/policies/{id}/run-for-site/` | POST | Targeted run for site(s) — `{"sites": [...]}`; creates and executes |
| `/policies/{id}/run-for-devices/` | POST | Targeted run for device(s) — `{"devices": [...]}`; creates and executes |
| `/rules/` | GET, POST | List and create rules (filter by `policy_id`, `engine`, `category`, `severity`, `is_active`) |
| `/rules/{id}/` | GET, PUT, PATCH, DELETE | Manage a rule |
| `/runs/` | GET, POST | List and create runs (filter by `policy_id`, `status`, `trigger`, `branch_id`, `has_branch`, `created_after/before`) |
| `/runs/{id}/` | GET, DELETE | Run status and summary; DELETE removes the run and cascades its results, findings and scores |
| `/runs/{id}/execute/` | POST | Execute a pending run |
| `/runs/validate-device/` | POST | Run all matching policies for a device |
| `/results/` | GET | Results (filter by `run_id`, `device_id`, `rule_id`, `status`, `severity`, `engine`) |
| `/findings/` | GET, POST | Findings (filter by `run_id`, `severity`, `status`, `category`, `policy_id`, `device_id`, `site_id`, `role_id`, `branch_id`, `is_new`, `latest`, `q`) |
| `/findings/{id}/` | GET, PUT, PATCH | Update finding status |
| `/findings/summary/` | GET | Finding counts by status, severity, category (respects filters) |
| `/findings/bulk-update-status/` | POST | `{"ids": [...], "status": "..."}` — bulk status change |
| `/findings/matrix/` | GET | Category × severity counts of open findings (`?site_id=`, `?role_id=`) |
| `/compliance/` | GET | Compliance scores (filter by `device_id`, `policy_id`, `run_id`, `site_id`, `role_id`) |
| `/compliance/summary/` | GET | Fleet score, device count, finding counts |
| `/compliance/trend/?days=N` | GET | Daily compliance scores over time |
| `/compliance/breakdown/?group_by=site\|role\|policy` | GET | Scores grouped by dimension (deleted policies keep their history under the snapshotted name) |
| `/compliance/distribution/` | GET | Histogram of device compliance scores |
| `/compliance/worst-performers/?limit=N` | GET | Lowest-scoring devices |
| `/policy-packs/` | GET | Bare array of packs (`slug`, `category`, `rule_count`, `engines`, `installed`, `installed_policy_ids`) |
| `/policy-packs/{slug}/` | GET | Pack details plus `policies[]` and `rules[]` |
| `/policy-packs/{slug}/install/` | POST | Install a pack (403 if its category is not enabled) |
| `/policy-packs/{slug}/uninstall/` | POST | Uninstall a pack |

All endpoints under `/api/plugins/validation/`.

---

## Version Notes

### Plugin 1.14.x (2026-09 – 2026-10)

- 1.14.0: NetBox 4.7 support (declares 4.4–4.7.x).
- 1.14.1: plugin-catalog metadata only.
- 1.14.2: rule `category` gains `routing`, `security`, `baseline` and `overlay` (used by the shipped packs alongside `addressing`, `redundancy`, `standards`, `completeness`, `security_intent`, `topology`, `reachability`, `power`, `infrastructure`).
- 1.14.3 (2026-10-06): no user-visible change.

### NetBox 4.7 (2026-09-02)

- Requires plugin **1.14.0+**. Checks evaluate NetBox data server-side, so check names and parameters are unchanged.
- What changes is the core REST API an agent uses to **remediate** findings: selection custom fields read as `{"value", "label"}` (write the raw value — relevant after `custom_field_populated` / `asset_documentation_complete` failures), `config_context` is present by default (omit it with `?omit=config_context` on 4.7.3+) on device reads (`?exclude=config_context` ignored — use `?fields=`), and services use `port_mappings` (`["tcp/443"]`) instead of `protocol` + `ports`. Failed bulk fixes return per-object errors by `index`. See [netbox-api-integration](../netbox-api-integration/SKILL.md).
- Branch-context runs on 4.7 need netbox-branching **1.2.x** (1.1.x for NetBox 4.4–4.6) — see [netbox-branching](../netbox-branching/SKILL.md).

### Plugin 1.12–1.13 (2026-08)

- 1.12.0: run lifecycle hardening (job timeout, orphaned-run reaper, run↔job linkage). 1.13.0: rules and policies deletable without losing run history.

---

## References

| Reference | When to Load |
|-----------|-------------|
| [references/api-patterns.md](references/api-patterns.md) | Need full API examples for policy/run/result lifecycle |
| [references/check-reference.md](references/check-reference.md) | Need check names, parameters, or engine details for all 95 checks |
| [references/policy-packs.md](references/policy-packs.md) | Need pack catalog, slugs, install/customize guidance, framework details |
