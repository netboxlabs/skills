---
name: netbox-validation
description: >
  NetBox Validation plugin — declarative policy-based validation of network infrastructure data
  and configuration compliance of intended configuration. Use when working with validation
  policies, compliance checks, hardening controls, config sources, findings, compliance scores,
  pre-change validation, or agentic safety nets.
license: Apache-2.0
---

# NetBox Validation

NetBox Validation continuously validates infrastructure against declarative policies using three engines — intent (NetBox data), configuration compliance (intended configuration), and graph analysis (infrastructure dependencies). It produces per-device results, aggregated findings, and compliance scores.

> **Your knowledge of NetBox Validation may be outdated.** Check names, API fields, policy pack contents, and engine capabilities evolve between releases. Prefer retrieval over pre-trained knowledge — the check catalog (`GET /checks/`) is the source of truth for check names and parameters.

**Current plugin:** 1.15.0 (configuration compliance: config sources, text rules, 82 hardening controls, source-of-truth parity, check catalog). NetBox 4.7 support arrived in 1.14.0; the plugin declares NetBox 4.4–4.7.x, and the docs require NetBox 4.5+ (NetBox Cloud). See [Version Notes](#version-notes).

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Validation docs | `https://netboxlabs.com/docs/validation/` | Full reference, engines, config sources, check catalog, policy packs |
| Validation REST API | `https://netboxlabs.com/docs/validation/api-reference/` | Endpoint details |
| Check catalog | `GET /api/plugins/validation/checks/` | Live check names, parameters, defaults, families, framework references |
| NetBox Platform MCP | If configured — 10 validation tools | Run validation, evaluate configs, query compliance, browse checks |
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
Policy (scope + rules + triggers + config source)
  -> Run (execution of a policy against devices; records config provenance + coverage)
    -> Results (per-device, per-check pass/fail/warning/error/skip)
      -> Findings (aggregated failures, actionable items)
        -> Compliance Scores (per-device, per-policy, 0-100%)
```

### Policy

A **Validation Policy** defines *what* to check and *where*. Scope filters: sites, regions, site groups, device roles, platforms, tags. Contains rules and controls engine toggles, the config source, triggers, and cron schedule.

### Rules

Each rule specifies:

| Field | Description |
|-------|-------------|
| `engine` | `intent`, `config`, or `graph` |
| `category` | Check grouping (addressing, redundancy, topology, security, etc.) |
| `check_name` | Specific check to run (e.g., `no_duplicate_ips`, `ssh_version_2`) |
| `severity` | `critical`, `high`, `medium`, `low`, or `info` |
| `parameters` | Check-specific config (thresholds, patterns, expected values) — merged over the check's defaults |

Rules can optionally override the policy scope with their own roles and platforms filters.

### Three Engines

| Engine | Checks | Evaluates | Data Source | Key Strength |
|--------|--------|-----------|-------------|--------------|
| Intent | 44 | NetBox data relationships | NetBox ORM | Fast, no external dependencies |
| Config | 142 | Intended configuration text | Config source (templates, Data Source, inline) | Configuration compliance, hardening, parity, reachability |
| Graph | 16 | Infrastructure dependency graph | NetBox ORM | Blast radius, failure domains |

All three can coexist in a single policy. Config and graph engines are available to Premium tier customers.

### Configuration Compliance — The Two-Step Narrative

The config engine answers "does the **intended configuration** satisfy the policy?" — *before it reaches the network*:

1. **Validate intent.** Configuration is rendered from NetBox Config Templates + config context, or produced by other tooling and stored in a repository. NetBox Validation evaluates that intended configuration on main and on branches, so a non-compliant change is caught in review.
2. **Verify the network.** NetBox Discovery and NetBox Assurance confirm that what runs in the network matches intent.

A **compliance finding** means the intended configuration violates a policy. Whether a live device differs from intent is an Assurance question — keep the two apart when explaining results.

The config engine has two layers. The **text layer** parses the configuration per **family** (`ios`, `nxos`, `iosxr`, `eos`, `junos`, `panos`, `fortios`, `asa`, `aoscx`) and runs text primitives, vendor-neutral hardening controls (resolved to each family's syntax through pattern tables) and the hostname parity check. The **model layer** (analysis service) builds a vendor-independent network model for routing/reachability/ACL questions and the remaining source-of-truth parity checks. Every result records `extra.layer`, `extra.family` and `extra.source`. Full detail: [references/config-compliance.md](references/config-compliance.md).

### Config Sources

| `config_source` | What is evaluated | Set up |
|---|---|---|
| `template` (default) | Config rendered per device from NetBox Config Templates | Assign templates (device/role/platform) |
| `datasource` | One file per device in a synced NetBox Data Source | `config_data_source` + `config_path_template` (default `{{ device.name }}.cfg`) |
| `inline` (run-level) | Text submitted in the request | `POST /policies/{id}/evaluate/` |
| `collected` | Configuration collected by NetBox Discovery | Not yet available; policies will not change |

### Run Statuses

| Status | Meaning |
|--------|---------|
| `pending` | Created, waiting for execution |
| `running` | Checks are executing |
| `passed` | All checks passed |
| `failed` | One or more checks failed |
| `error` | Engine error during execution |

Since plugin 1.12.0 each run is linked to its NetBox background job, bounded by a job timeout, and orphaned runs (worker lost mid-run) are reaped rather than left in `running`.

Each run exposes `job_id` (the core background Job), `skipped_checks`, `config_source` and `config_provenance`. The plugin settings `job_timeout` (default 3600s) and `run_retention_days` (default 180; `<= 0` disables) govern lifecycle: completed runs older than the window are pruned by the hourly housekeeping job, which never prunes in-flight runs and always keeps each policy's most recent run. Bulk cleanup from an agent should prefer the MCP `prune_validation_runs` tool (dry-run by default) over looping `DELETE /runs/{id}/`.

### Result Statuses

| Status | Meaning | Score | Finding |
|--------|---------|-------|---------|
| `pass` | Requirement satisfied | passed | no |
| `fail` | Requirement violated | failed | yes |
| `warning` | Advisory (parse warnings, coverage, unused structures) | counts as **passed** | no |
| `error` | Check could not run | failed | yes |
| `skip` | Not evaluated (no config, family unresolved/unsupported, control not applicable, engine disabled) | **excluded** | no |

Warnings never fail anything. Skips never lower a score — a device with no configuration is a visible gap (`extra.reason`, run `skipped_checks`, `GET /runs/{id}/coverage/`), not a false pass or fail. Fleet-scoped model checks produce one result per run with `device: null`.

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

Skipped checks excluded; warnings count as passed. Scores are per (device, policy, run) and tracked over time.

---

## Quick Reference — Common Workflows

### Create and Execute a Run

Creating a run does NOT execute it. Two-step process:

```bash
RUN_ID=$(curl -s -X POST "$NETBOX_URL/api/plugins/validation/runs/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual"}' | python -c "import sys,json; print(json.load(sys.stdin)['id'])")
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/$RUN_ID/execute/" -H "Authorization: Bearer $NETBOX_TOKEN"   # synchronous
```

The execute endpoint is **synchronous** — it returns results when the run completes. Do not poll.

### Install a Policy Pack

```bash
curl "$NETBOX_URL/api/plugins/validation/policy-packs/" -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/management-plane-security/install/" -H "Authorization: Bearer $NETBOX_TOKEN"
```

### Configure a Config Source and Evaluate Inline

```bash
# Intended configuration files in a Data Source, one per device
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/12/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"config_source": "datasource", "config_data_source": 3,
       "config_path_template": "{{ device.site.slug }}/{{ device.name }}.cfg"}'

# Evaluate configuration text from a CI pipeline (keys = device names or IDs; text is not stored)
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/12/evaluate/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"configs": {"leaf1": "hostname leaf1\n..."}, "wait": true, "origin": "ci:build-123"}'

# What did the run actually evaluate? Provenance, families, missing devices, coverage
curl "$NETBOX_URL/api/plugins/validation/runs/920/coverage/" -H "Authorization: Bearer $NETBOX_TOKEN"
```

### Discover Checks Before Writing Rules

```bash
curl "$NETBOX_URL/api/plugins/validation/checks/?engine=config&layer=text" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/checks/?group=Hardening" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/checks/config_each_block/" -H "Authorization: Bearer $NETBOX_TOKEN"
```

### Query Compliance Scores and Findings

```bash
curl "$NETBOX_URL/api/plugins/validation/compliance/" -H "Authorization: Bearer $NETBOX_TOKEN"                                  # fleet-wide latest
curl "$NETBOX_URL/api/plugins/validation/compliance/?device_id=10&ordering=-measured_at" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/findings/?latest=true&status=open&severity=critical" -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X PATCH "$NETBOX_URL/api/plugins/validation/findings/42/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" -d '{"status": "acknowledged"}'
```

### Run Against a Branch / Targeted Runs

```bash
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" -d '{"policy": 1, "trigger": "manual", "branch_id": 5}'
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" -d '{"policy": 1, "trigger": "api", "target_sites": [9]}'
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/validate-device/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" -d '{"device": 101}'
```

The engines read rules from main and execute checks in branch context, seeing branch-overlay data. With the Config Templates source, the proposed configuration is rendered from the branch.

### YAML Import/Export and Clone

```bash
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/export/" -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/import/" -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" -d @my-policy.yaml
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/clone/" -H "Authorization: Bearer $NETBOX_TOKEN"
```

YAML carries `config_source` and `config_path_template` but not `config_data_source` — PATCH it after import.

---

## Policy Management

### Policy Scoping

Policies scope devices via any combination of **sites, regions, site groups, device roles, platforms, tags** (`scope_sites`, `scope_regions`, `scope_site_groups`, `scope_roles`, `scope_platforms`, `scope_tags`). Scope fields are AND-combined. An empty scope means all active devices.

### Engine Toggles and Config Source

| Policy Field | Default | Effect |
|-------------|---------|--------|
| `enable_config_engine` | `false` | Enable configuration compliance checks for this policy |
| `enable_graph_engine` | `false` | Enable graph resilience checks for this policy |
| `config_source` | `template` | `template` or `datasource` — where config rules get their text |
| `config_data_source` | -- | Data Source ID; required when `config_source` is `datasource` |
| `config_path_template` | `{{ device.name }}.cfg` | Jinja2 path to each device's file in the Data Source |

Intent checks always run. Config and graph checks require their engine to be enabled on the policy AND available at the deployment level.

### Writing Config Rules

Four kinds of `engine: config` checks. Always read `GET /checks/{name}/` for exact parameter names and defaults; examples and parameter tables are in [references/config-compliance.md](references/config-compliance.md).

| Kind | Examples | Parameters |
|------|----------|------------|
| Text primitives (9) | `config_contains`, `config_not_contains`, `config_section_present`/`absent`, `config_each_block` (`block`, `when`, `exclude`, `require`, `forbid`), `config_value` (`pattern`, `op`, `value`, `value_from`), `config_section_allowlist`, `config_banner_contains`, `config_unparsed_lines` | Generic, parameterised, all families |
| Hardening controls (82) | `ssh_version_2`, `telnet_disabled`, `aaa_enabled`, `vty_exec_timeout`, `snmp_v3_priv_required`, `logging_hosts_min_count`, `ntp_authentication_enabled`, `bgp_authentication_configured`, `urpf_enabled_on_edge`, `bpduguard_on_access_ports`, `dhcp_snooping_enabled` | Mostly none; thresholds and `edge`/`access_when` selectors on some; `framework_refs` on all |
| Source-of-truth parity (17) | `hostname_matches_netbox`, `interface_descriptions_match_netbox`, `interface_ips_match_ipam`, `vrfs_match_netbox`, `vlans_match_netbox`, `bgp_neighbors_match_netbox`, `ntp_servers_match_context` | `expected` = NetBox, `actual` = configuration |
| Model layer (34) | `config_parse_status`, `bgp_sessions`, `duplicate_ips`, `acl_reachability`, `assert_traffic_blocked`, `differential_reachability` | See [references/check-reference.md](references/check-reference.md) |

Hardening pattern tables are per family and still expanding: a control with no pattern for the device's family returns `skip` ("no pattern for family"), never `fail` — check `families` in the catalog and use text primitives where needed. Map custom platform slugs with the plugin setting `platform_families: {"<slug>": "<family>"}`.

### Policy Packs

Pre-built policy packs provide one-click installation:

- **18 starter packs** — addressing, cabling, data quality, naming, redundancy, security intent, leaf/spine baselines, power/network/full resilience, BGP attributes, pre-change config, and five configuration-compliance packs: **Management Plane Security** (36), **Layer 2 Hardening** (19), **Routing Protocol Security** (16), **Source-of-Truth Parity** (17), **Intended Config Baseline** (19)
- **12 compliance frameworks** (Premium; `enabled_policy_packs` must include `"compliance"`) — CLOS Fabric, TIA-942, NIS2/DORA, NIST 800-53, NERC CIP, PCI-DSS, MANRS, ISO 27001, HIPAA Security Rule (2026) — all rebuilt on the check catalog with hardening controls — plus **CIS Benchmarks — Network Devices** (64), **DISA STIG — Routers and Switches** (64), **NSA Network Infrastructure Security** (57). 525 rules total.

Installed packs create regular policies and rules. After installation, customize scope, config source, parameters, triggers, and schedule. See [references/policy-packs.md](references/policy-packs.md).

### Delete Rules and Policies

Since plugin 1.13.0, `DELETE /rules/{id}/` and `DELETE /policies/{id}/` preserve historical run results. Results keep `rule_name` and `check_name` snapshots (the rule's parameters are kept in `extra.rule_parameters`), and runs and compliance scores keep `policy_name`; after a deletion the `rule` / `policy` FK fields are `null`, so read the `*_name` fields rather than assuming the FK is set. Policy-pack uninstall is no longer refused when runs exist — the pack's policies are removed and their run history is retained under the snapshotted policy name. Deleting a **run** (`DELETE /runs/{id}/`) is the one destructive path: it cascades the run's results, findings and compliance scores.

---

## Validation Triggers

### Manual

Create a run via API or click "Run Now" in the UI. Two-step: create run, then execute.

### Branch Merge

Set `trigger_on_branch_merge: true` on the policy. Validation runs automatically when a NetBox Branching branch is merged to main. Catches regressions in the merged data.

### Change Request Submit

Set `trigger_on_cr_submit: true` on the policy. Validation runs when a NetBox Changes change request is submitted for review. The run targets the CR's branch, providing pre-merge safety.

### Scheduled

Set a cron expression on the policy's `schedule` field — `0 2 * * *` (daily 02:00), `0 */6 * * *` (every 6 hours), `0 0 * * 0` (weekly, Sunday midnight). Scheduled runs target main (no branch context).

### Inline (API)

`POST /policies/{id}/evaluate/` creates and (with `wait: true`) executes a run with `config_source: inline` and `trigger: api`. Only config rules run; intent and graph rules are skipped for that run. Pair it with a scheduled or CR-triggered run of the same policy for full coverage.

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

### Score Dimensions

Compliance scores can be queried by **device** (across policies), **policy** (across devices), and **time** (7d/30d/90d trend via the compliance dashboard).

### Finding Aggregation

Findings group related failures into actionable items. For example, if `no_duplicate_ips` fails for 5 devices, one finding groups all 5. Graph engine findings group by failure point (one finding per power feed or shared failure domain). Compliance findings on hardening controls carry the control's `framework_refs`, so each can be traced to the requirement it serves.

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
2. Review detail: affected devices, `expected` vs `actual`, `remediation`; for config results read `extra.missing` / `violations` / `mismatches`
3. For graph findings, check blast radius card; for config runs, check `GET /runs/{id}/coverage/` for devices that had nothing to evaluate
4. Acknowledge findings you plan to fix
5. Fix the underlying issue — NetBox data for intent findings; the Config Template, config context, or Data Source file for compliance findings
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

### CI Pipeline for Intended Configuration

```
1. Pipeline renders or collects intended configs for the changed devices
2. POST /policies/{id}/evaluate/ with {"configs": {...}, "wait": true, "origin": "ci:<build>"}
3. Gate on run.status; read failing results' extra.missing / violations / mismatches + remediation
4. Fix, re-evaluate, then push the change (or open the NetBox change request)
```

### Branch Context

When running validation against a branch, pass `branch_id` on the run. The engines read rules from main and execute checks in branch context (seeing proposed additions, modifications, deletions). With the Config Templates source the proposed configuration is rendered from the branch; the main render is the reference for the differential checks (`differential_reachability`, `routing_changes`).

### Agentic Integration via MCP

The NetBox Labs Platform MCP Server provides 10 validation tools:

| Tool | Purpose |
|------|---------|
| `run_validation` | Create + execute a run for a policy, optionally targeting a branch or devices |
| `get_compliance_analytics` | Compliance scores, trends, score-by-dimension |
| `get_findings_analytics` | Findings filtered by severity, status, category, policy |
| `import_validation_policy` | Import policy from YAML |
| `clone_validation_policy` | Clone policy with all rules and scoping |
| `prune_validation_runs` | Bulk-delete old completed runs (dry-run unless confirmed) |
| `list_validation_checks` | Check catalog — filter by `engine`, `layer`, `group`, `family`, `query` |
| `get_validation_check` | One check's parameters, defaults, families, `framework_refs` — read before writing a rule |
| `get_validation_run_coverage` | Config source provenance, families, missing devices and coverage for a run |
| `evaluate_device_configs` | Inline evaluation of `{device: config text}` against a policy (`policy_id`, `configs`, `origin`, `wait`; ≤ 50 devices / 1 MB) |

Agents that prefer direct API access can use the REST endpoints with bearer token authentication.

### Agentic Workflow Pattern

```
Agent creates branch -> makes changes -> runs validation (and evaluates rendered configs)
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
| Guess config check parameter names | Unknown parameters are ignored (defaults apply); missing required ones (`block`, `pattern`, `section`) produce `error` | Read `GET /checks/{name}/` first |
| Expect hardening controls to fail on an unsupported family | Controls skip ("no pattern for family") where no pattern exists | Check `families` in the catalog; use text primitives for those devices |
| Treat `skip` or `warning` as failures | Skips are excluded from scores; warnings count as passed | Read `skipped_checks` and `/runs/{id}/coverage/` to find real gaps |
| Describe compliance findings as the live network being wrong | Config rules evaluate *intended* configuration, not device state | Say the intended configuration violates the policy; Assurance verifies the network |
| Call `evaluate` with unknown device names | 404; devices must exist for scoping and parity | Create the device in NetBox first |
| Expect target scoping to widen policy scope | Target fields AND-combine with policy scope | Targets can only narrow — devices outside policy scope are excluded |
| Poll for run completion after execute | Execute is synchronous — returns when done | Read the response directly; no polling needed |
| Run graph checks with device targeting expecting savings | Graph engine loads all site devices regardless | Use site-scoped runs for graph-heavy policies |

---

## API Endpoints Summary

| Endpoint | Methods | Purpose |
|----------|---------|---------|
| `/policies/` | GET, POST | List and create policies |
| `/policies/{id}/` | GET, PUT, PATCH, DELETE | Manage a policy (incl. `config_source`, `config_data_source`, `config_path_template`) |
| `/policies/{id}/clone/` | POST | Clone with all rules and scoping |
| `/policies/{id}/export/` | POST | Export as YAML |
| `/policies/import/` | POST | Import from YAML |
| `/policies/export-all/` | POST | Export all policies |
| `/policies/{id}/run-for-site/` | POST | Targeted run for site(s) |
| `/policies/{id}/run-for-devices/` | POST | Targeted run for device(s) |
| `/policies/{id}/evaluate/` | POST | Evaluate the policy's config rules against configuration text submitted inline (`configs`, `wait`, `origin`) |
| `/rules/` | GET, POST | List and create rules |
| `/rules/{id}/` | GET, PUT, PATCH, DELETE | Manage a rule |
| `/runs/` | GET, POST | List and create runs |
| `/runs/{id}/` | GET, DELETE | Run status and summary (`skipped_checks`, `config_source`, `config_provenance`); DELETE cascades results, findings and scores |
| `/runs/{id}/execute/` | POST | Execute a pending run |
| `/runs/{id}/coverage/` | GET | Config source provenance, families, missing devices and per-device coverage |
| `/runs/{id}/matrix/?engine=` | GET | Compact device × check status matrix (`checks`, `devices[].cells`, `fleet`) |
| `/runs/validate-device/` | POST | Run all matching policies for a device |
| `/results/` | GET | Results (filter by run, device, status, check_name, rule_name, device_name) |
| `/findings/` | GET, POST | Findings (filter by run, severity, status, category, site, role, `latest`) |
| `/findings/{id}/` | GET, PUT, PATCH | Update finding status |
| `/findings/summary/` | GET | Finding counts by status, severity, category (respects filters) |
| `/findings/bulk-update-status/` | POST | `{"ids": [...], "status": "..."}` — bulk status change |
| `/compliance/` | GET | Compliance scores (filter by device, policy) |
| `/compliance/summary/` | GET | Fleet score, device count, finding counts |
| `/compliance/trend/?days=N` | GET | Daily compliance scores over time |
| `/compliance/breakdown/?group_by=site\|role\|policy` | GET | Scores grouped by dimension (deleted policies keep their history under the snapshotted name) |
| `/compliance/distribution/` | GET | Histogram of device compliance scores |
| `/compliance/worst-performers/?limit=N` | GET | Lowest-scoring devices |
| `/checks/` | GET | Catalog of every built-in check (filter by `engine`, `layer`, `group`, `category`, `family`) |
| `/checks/{name}/` | GET | One check: parameters, defaults, families, sources, `requires_proposed`, `framework_refs` |
| `/policy-packs/` | GET | List available policy packs |
| `/policy-packs/{slug}/` | GET | Pack details with rule list |
| `/policy-packs/{slug}/install/` | POST | Install a pack |
| `/policy-packs/{slug}/uninstall/` | POST | Uninstall a pack |

All endpoints under `/api/plugins/validation/`.

---

## Version Notes

### Plugin 1.15 — Configuration Compliance (2026-10)

- Config engine rebuilt around **config sources** (Config Templates, Data Source, inline API), a **text layer** (9 primitives, 82 vendor-neutral hardening controls with framework references, `hostname_matches_netbox`) and a **model layer** (34 checks plus 16 source-of-truth parity checks). Config checks: 35 → 142.
- New endpoints: `POST /policies/{id}/evaluate/`, `GET /runs/{id}/coverage/`, `GET /runs/{id}/matrix/`, `GET /checks/`, `GET /checks/{name}/`. New policy fields `config_source`, `config_data_source`, `config_path_template`; run fields `skipped_checks`, `config_source`, `config_provenance`.
- Result semantics: `warning` counts as passed, `skip` is excluded from scores, fleet-scoped checks yield one result per run.
- Packs: 4 new starter packs + Intended Config Baseline (renamed, slug `config-analysis`); 3 new framework packs (CIS, DISA STIG, NSA); regulatory packs rebuilt (525 framework rules).
- Plugin setting `platform_families` maps platform slugs to configuration families.

### NetBox 4.7 (2026-09-02)

- Requires plugin **1.14.0+**. Checks evaluate NetBox data server-side, so check names and parameters are unchanged.
- What changes is the core REST API an agent uses to **remediate** findings: selection custom fields read as `{"value", "label"}` (write the raw value — relevant after `custom_field_populated` / `asset_documentation_complete` failures), `config_context` is always present on device reads (`?exclude=config_context` ignored — use `?fields=`), and services use `port_mappings` (`["tcp/443"]`) instead of `protocol` + `ports`. Failed bulk fixes return per-object errors by `index`. See [netbox-api-integration](../netbox-api-integration/SKILL.md).
- Branch-context runs on 4.7 need netbox-branching **1.2.x** (1.1.x for NetBox 4.4–4.6) — see [netbox-branching](../netbox-branching/SKILL.md).

### Plugin 1.12–1.13 (2026-08)

- 1.12.0: run lifecycle hardening (job timeout, orphaned-run reaper, run↔job linkage). 1.13.0: rules and policies deletable without losing run history.

---

## References

| Reference | When to Load |
|-----------|-------------|
| [references/config-compliance.md](references/config-compliance.md) | Configuring a config source, writing text/hardening/parity rules, result `extra` keys, coverage/matrix/catalog responses, skip reasons |
| [references/api-patterns.md](references/api-patterns.md) | Need full API examples for policy/run/result lifecycle |
| [references/check-reference.md](references/check-reference.md) | Need check names, parameters, or engine details for all 202 checks |
| [references/policy-packs.md](references/policy-packs.md) | Need pack catalog, slugs, install/customize guidance, framework details |
