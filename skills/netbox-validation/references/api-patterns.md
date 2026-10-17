# API Patterns — NetBox Validation

All endpoints under `/api/plugins/validation/`. Auth: `Authorization: Bearer <token>`.

**Response conventions.** Choice fields — `config_source`, `trigger`, `status`, `engine`, `category`, `severity` — read back as `{"value": "...", "label": "..."}` on policies, rules, runs and results; writes take the bare string. Exceptions: `config_source` on the run detail and on `/runs/{id}/coverage/`, and `status`/`trigger` inside a policy's `latest_run` summary, are bare strings. Unknown body fields are silently ignored (no 400) — read the object back after a write.

## Policy CRUD

```bash
# List policies
curl "$NETBOX_URL/api/plugins/validation/policies/" -H "Authorization: Bearer $NETBOX_TOKEN"

# Create a policy (config engine on, default config source = Config Templates).
# Scope fields are sites / regions / site_groups / roles / platforms / device_tags — lists of IDs.
# There is no "scope_" prefix: "scope_sites" is silently ignored and yields an UNSCOPED policy.
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"name": "Leaf Baseline", "is_active": true, "enable_config_engine": true,
    "trigger_on_cr_submit": true, "schedule": "0 2 * * *",
    "sites": [1, 2], "roles": [3], "device_tags": [7]}'
# Always confirm the scope landed: the response (or GET /policies/{id}/) must show
# "sites": [1, 2], "roles": [3] — empty lists mean every active device is in scope.
# trigger_on_cr_submit defaults to true on create; trigger_on_branch_merge to false.

# Update / Delete / Clone
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/1/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"enable_graph_engine": true}'

curl -X DELETE "$NETBOX_URL/api/plugins/validation/policies/1/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/clone/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## Rule Management

```bash
# List rules for a policy
curl "$NETBOX_URL/api/plugins/validation/rules/?policy_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Create an intent rule
curl -X POST "$NETBOX_URL/api/plugins/validation/rules/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "name": "Minimum Uplinks", "engine": "intent",
    "category": "redundancy", "check_name": "min_cabled_uplinks",
    "severity": "critical", "is_active": true, "parameters": {"min_uplinks": 4}}'

# Create a config rule (text primitive) — look up parameters first: GET /checks/config_each_block/
curl -X POST "$NETBOX_URL/api/plugins/validation/rules/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "name": "VTY SSH only", "engine": "config", "category": "security",
    "check_name": "config_each_block", "severity": "critical",
    "parameters": {"block": "^line vty", "require": ["^transport input ssh$", "^access-class \\S+ in"], "forbid": ["telnet"]}}'

# Create a config rule (hardening control, no parameters)
curl -X POST "$NETBOX_URL/api/plugins/validation/rules/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "name": "SSH v2", "engine": "config", "category": "security",
    "check_name": "ssh_version_2", "severity": "high"}'

# Update a rule (rule-level scope overrides are "roles" and "platforms", ID lists)
curl -X PATCH "$NETBOX_URL/api/plugins/validation/rules/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"parameters": {"min_uplinks": 2}, "severity": "high", "roles": [3]}'

# Delete a rule (plugin 1.13+: prior runs keep its name/outcome as "(deleted rule)")
curl -X DELETE "$NETBOX_URL/api/plugins/validation/rules/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## Check Catalog

```bash
# Every built-in check across the three engines
curl "$NETBOX_URL/api/plugins/validation/checks/" -H "Authorization: Bearer $NETBOX_TOKEN"

# Config text-layer checks only / hardening controls only / checks valid for a family
curl "$NETBOX_URL/api/plugins/validation/checks/?engine=config&layer=text" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/checks/?group=Hardening" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/checks/?engine=config&family=junos" -H "Authorization: Bearer $NETBOX_TOKEN"

# One check: parameters schema, default_params, families, plane, sources, requires_proposed, framework_refs
curl "$NETBOX_URL/api/plugins/validation/checks/vty_exec_timeout/" -H "Authorization: Bearer $NETBOX_TOKEN"
# e.g. GET /checks/vtp_transparent_or_off/ -> "families": ["ios", "nxos"], "plane": "data"
```

Filters: `engine`, `layer` (`text` | `model`), `group`, `category`, `family`. Response: `{"count": N, "results": [...]}` (218 checks: 44 intent, 158 config, 16 graph). On hardening controls `families` comes from the pattern tables — a device whose family is absent produces `skip` (`extra.reason` "not applicable" or "no pattern for family"), so check it before adding the rule.

## Configuration Compliance

```bash
# Point a policy at intended configuration files in a NetBox Data Source (one file per device)
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/12/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"config_source": "datasource", "config_data_source": 3,
       "config_path_template": "{{ device.site.slug }}/{{ device.name }}.cfg"}'

# Back to rendered Config Templates (default)
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/12/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"config_source": "template"}'

# Evaluate configuration text inline (CI pipeline, partner tooling). Keys = device names or IDs.
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/12/evaluate/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"configs": {"leaf1": "hostname leaf1\n...", "42": "hostname leaf2\n..."},
       "wait": true, "origin": "ci:build-123"}'
# wait=true  -> 200 {"run": {...}, "results": [...]}
# wait=false -> 202 run (poll GET /runs/{id}/)

# Provenance and coverage for a run
curl "$NETBOX_URL/api/plugins/validation/runs/920/coverage/" -H "Authorization: Bearer $NETBOX_TOKEN"

# Device x check status matrix (optionally one engine)
curl "$NETBOX_URL/api/plugins/validation/runs/920/matrix/?engine=config" -H "Authorization: Bearer $NETBOX_TOKEN"
```

- Policies expose `config_source` (`template` | `datasource`), `config_data_source`, `config_data_source_name`, `config_path_template`.
- Runs expose `config_source` (bare string; adds `inline` for evaluate runs), `config_provenance`, `skipped_checks`.
- Results carry `expected`, `actual`, `remediation` (where the pattern table provides it) and `extra` with `layer`, `family`, `source` and check-specific detail: `missing`, `extra` (hardening: `[{line, path, text}]`), `violations`, `violating`, `mismatches`, `mode` + `conditions[]` on compound controls, `reason` on skips, and `rule_parameters` only when the rule has parameters.

See [config-compliance.md](config-compliance.md) for response shapes and rule-writing guidance.

## Run Lifecycle

```bash
# Create a run — policy and trigger are required; status defaults to "pending"; NOT executed yet
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual", "status": "pending"}'

# Execute (synchronous — returns when complete, no polling needed)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/1/execute/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Check status / List recent runs
curl "$NETBOX_URL/api/plugins/validation/runs/1/" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/runs/?policy_id=1&ordering=-created" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## Run History and Retention

```bash
# Delete a run — cascades its results, findings and compliance scores (plugin 1.13+)
curl -X DELETE "$NETBOX_URL/api/plugins/validation/runs/1/" -H "Authorization: Bearer $NETBOX_TOKEN"
```

- Deleting a rule or policy never removes history (see Rule Management above). Results carry `rule_name` / `check_name`; runs and scores carry `policy_name`; the `rule` / `policy` FKs are `null` after deletion.
- Completed runs are pruned after `run_retention_days` (plugin setting, default 180). In-flight runs are never pruned and each policy's most recent run is always kept.
- Each run carries `job_id`. A run left in `running` by a worker crash is reconciled to `error` by the hourly housekeeping job; `job_timeout` (default 3600s) bounds a hung run.

## Targeted Runs

```bash
# Target sites / devices / tags
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "api", "status": "pending", "target_sites": [9]}'

curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "api", "status": "pending", "target_devices": [101, 102, 103]}'

# Convenience endpoints (create + enqueue automatically)
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/run-for-site/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"sites": [9]}'

curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/run-for-devices/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"devices": [101, 102, 103]}'

# Validate one device across all matching policies
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/validate-device/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"device": 101}'
```

## Branch-Context Runs

```bash
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual", "status": "pending", "branch_id": 5}'
# Then execute as normal. Rules read from main; checks run in branch context.
# With the Config Templates source, the proposed configuration is rendered from the branch.
```

## Results and Findings

```bash
# Results: filter by run, device, status, check_name, rule_name, device_name
curl "$NETBOX_URL/api/plugins/validation/results/?run_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/results/?status=fail&device_id=10" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/results/?run_id=1&check_name=ssh_version_2" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Findings: filter by severity, status, run, category, site, role; latest=true for current findings only
curl "$NETBOX_URL/api/plugins/validation/findings/?latest=true&severity=critical&status=open" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Update finding status
curl -X PATCH "$NETBOX_URL/api/plugins/validation/findings/42/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"status": "acknowledged"}'

# Bulk status change / counts by status, severity, category (both honor the findings filters)
curl -X POST "$NETBOX_URL/api/plugins/validation/findings/bulk-update-status/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"ids": [42, 43, 44], "status": "acknowledged"}'
curl "$NETBOX_URL/api/plugins/validation/findings/summary/?status=open" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

Remediating a finding means writing to core NetBox objects — on NetBox **4.7** selection custom fields read back as `{"value", "label"}` (write the raw value), services take `port_mappings`, and failed bulk writes report per-object errors by `index`. See [netbox-api-integration](../../netbox-api-integration/SKILL.md). Remediating a **compliance finding** means changing the intended configuration — the Config Template, config context, or the file in the Data Source — then re-running.

## Compliance Scores

```bash
curl "$NETBOX_URL/api/plugins/validation/compliance/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/?device_id=10&ordering=-measured_at" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/?policy_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Fleet analytics (current state: latest completed run per active policy; inline evaluations never count)
curl "$NETBOX_URL/api/plugins/validation/compliance/summary/?target=80" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/trend/?days=30" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/breakdown/?group_by=platform" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
# Configuration compliance: coverage, hardening/framework posture, source-of-truth drift
curl "$NETBOX_URL/api/plugins/validation/compliance/config-coverage/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/posture/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## Policy Packs

```bash
# List — a bare JSON array (no count/results). Each entry: slug, name, description, category,
# policy_count, rule_count, engines, families (families where >= 50% of the pack's config rules
# apply; null for packs without config rules), family_coverage ({family: share}), installed,
# installed_policy_ids
curl "$NETBOX_URL/api/plugins/validation/policy-packs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
# Details — same fields plus policies[] and rules[] (name, engine, category, check_name, severity, description)
curl "$NETBOX_URL/api/plugins/validation/policy-packs/management-plane-security/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/management-plane-security/install/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/management-plane-security/uninstall/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## YAML Import/Export

```bash
# Export single / all
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/export/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/export-all/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Import
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/import/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d @my-policy.yaml
```

### YAML Format

```yaml
name: Leaf Baseline
description: Intent + configuration compliance for leaf switches
enable_config_engine: true
enable_graph_engine: false
config_source: template          # or datasource (then PATCH config_data_source after import)
config_path_template: "{{ device.name }}.cfg"
trigger_on_cr_submit: true
schedule: "0 2 * * *"
rules:
  - name: No Duplicate IPs
    engine: intent
    category: addressing
    check_name: no_duplicate_ips
    severity: critical
  - name: Minimum Uplinks
    engine: intent
    category: redundancy
    check_name: min_cabled_uplinks
    severity: critical
    parameters:
      min_uplinks: 2
  - name: SSH Version 2 Only
    engine: config
    category: security
    check_name: ssh_version_2
    severity: high
  - name: NTP servers from config context
    engine: config
    category: standards
    check_name: config_value
    severity: medium
    parameters:
      pattern: '^ntp server (\S+)'
      op: superset_of
      value_from: "config_context:ntp.servers"
```

Export carries `config_source` and `config_path_template` but not `config_data_source` (environment-specific ID).
