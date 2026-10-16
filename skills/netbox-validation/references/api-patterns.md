# API Patterns — NetBox Validation

All endpoints under `/api/plugins/validation/`. Auth: `Authorization: Bearer <token>`.

**Response conventions.** Choice fields — `engine`, `category`, `severity` on rules; `trigger`, `status` on runs; `status`, `severity` on results; `severity`, `category`, `status` on findings — read back as `{"value": "...", "label": "..."}`; writes take the bare string. The `status`/`trigger` inside a policy's `latest_run` summary are bare strings. Unknown body fields are silently ignored (no 400) — read the object back after a write.

## Policy CRUD

```bash
# List policies
curl "$NETBOX_URL/api/plugins/validation/policies/" -H "Authorization: Bearer $NETBOX_TOKEN"

# Create a policy.
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
# The response also carries rules_count, next_scheduled_run and latest_run {id, status, score, trigger, created}.

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
# List rules for a policy (filters: policy_id, engine, category, severity, is_active, q)
curl "$NETBOX_URL/api/plugins/validation/rules/?policy_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Create a rule
curl -X POST "$NETBOX_URL/api/plugins/validation/rules/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "name": "Minimum Uplinks", "engine": "intent",
    "category": "redundancy", "check_name": "min_cabled_uplinks",
    "severity": "critical", "is_active": true, "parameters": {"min_uplinks": 4}}'

# Update a rule (rule-level scope overrides are "roles" and "platforms", ID lists)
curl -X PATCH "$NETBOX_URL/api/plugins/validation/rules/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"parameters": {"min_uplinks": 2}, "severity": "high", "roles": [3]}'

# Delete a rule (plugin 1.13+: prior runs keep its name/outcome as "(deleted rule)")
curl -X DELETE "$NETBOX_URL/api/plugins/validation/rules/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

Rule `category` accepts `addressing`, `redundancy`, `standards`, `completeness`, `security_intent`, `topology`, `reachability`, `power`, `infrastructure`, `routing`, `security`, `baseline`, `overlay`.

## Run Lifecycle

```bash
# Create a run — policy, trigger AND status are required (400 without "status"); NOT executed yet
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual", "status": "pending"}'

# Execute (synchronous — returns the completed run; only pending/error runs can be executed)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/1/execute/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Check status / List recent runs
curl "$NETBOX_URL/api/plugins/validation/runs/1/" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/runs/?policy_id=1&ordering=-created" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

Run fields: `policy`, `policy_name`, `trigger`, `status`, `branch_id`, `branch_name`, `change_request_id`, `job_id`, `started_at`, `completed_at`, `devices`, `target_sites`, `target_devices`, `target_tags`, `total_checks`, `passed_checks`, `failed_checks`, `warning_checks`, `error_checks`, `score`. Note that `warning_checks` on a run counts **skipped** results. Run filters: `policy_id`, `status`, `trigger`, `branch_id`, `has_branch`, `change_request_id`, `created_after`, `created_before`.

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
# Target sites / devices / tags (all ID lists)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "api", "status": "pending", "target_sites": [9]}'

curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "api", "status": "pending", "target_devices": [101, 102, 103]}'

# Convenience endpoints (create + execute in one call; 201 with the finished run)
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/run-for-site/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"sites": [9]}'

curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/run-for-devices/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"devices": [101, 102, 103]}'

# Validate one device across all matching active policies (device must be status=active)
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
```

## Results and Findings

```bash
# Results: filter by run_id, device_id, rule_id, status, severity, engine
curl "$NETBOX_URL/api/plugins/validation/results/?run_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/results/?status=fail&device_id=10" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/results/?run_id=1&engine=graph" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Findings: filter by severity, status, run_id, category, policy_id, device_id, site_id, role_id,
# branch_id, is_new, q (title/description/remediation); latest=true for the latest run per policy
curl "$NETBOX_URL/api/plugins/validation/findings/?latest=true&severity=critical&status=open" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/findings/?latest=true&q=cable&site_id=9" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Update finding status
curl -X PATCH "$NETBOX_URL/api/plugins/validation/findings/42/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"status": "acknowledged"}'

# Bulk status change / counts by status, severity, category (summary honors the findings filters)
curl -X POST "$NETBOX_URL/api/plugins/validation/findings/bulk-update-status/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"ids": [42, 43, 44], "status": "acknowledged"}'
# -> {"updated": 3}
curl "$NETBOX_URL/api/plugins/validation/findings/summary/?status=open" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
# -> {"by_status": {...}, "by_severity": {...}, "by_category": {...}, "total": N}

# Category x severity matrix of open findings
curl "$NETBOX_URL/api/plugins/validation/findings/matrix/?site_id=9" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

Result fields: `run`, `rule`, `rule_name`, `check_name`, `device`, `device_name`, `status`, `severity`, `expected`, `actual`, `message`, `remediation`, `extra`. Finding fields: `run`, `title`, `display_title`, `description`, `severity`, `category`, `results`, `remediation`, `status`, `finding_key`, `is_new`, `recurrence_count`, `first_seen`, `affected_devices` (`[{id, name}]`), `has_blast_radius`, `policy_id`, `policy_name`.

Remediating a finding means writing to core NetBox objects — on NetBox **4.7** selection custom fields read back as `{"value", "label"}` (write the raw value), services take `port_mappings`, and failed bulk writes report per-object errors by `index`. See [netbox-api-integration](../../netbox-api-integration/SKILL.md).

## Compliance Scores

```bash
curl "$NETBOX_URL/api/plugins/validation/compliance/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/?device_id=10&ordering=-measured_at" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/?policy_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Dashboard aggregates (site_id / role_id narrow all of them)
curl "$NETBOX_URL/api/plugins/validation/compliance/summary/" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/trend/?days=30" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/breakdown/?group_by=policy" -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/worst-performers/?limit=10" -H "Authorization: Bearer $NETBOX_TOKEN"
```

Score rows carry `device`, `device_name`, `site_id`/`site_name`, `role_id`/`role_name`, `policy`, `policy_name`, `run`, `score`, `total_checks`, `passed_checks`, `failed_checks`, `measured_at`.

## Policy Packs

```bash
# List — a bare JSON array (no count/results). Each entry: slug, name, description,
# category (starter|compliance), policy_count, rule_count, engines, installed, installed_policy_ids.
# Compliance packs are listed only when the plugin setting enabled_policy_packs includes "compliance".
curl "$NETBOX_URL/api/plugins/validation/policy-packs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
# Details — same fields plus policies[] and rules[] (name, engine, category, check_name, severity, description)
curl "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
# Install -> 201 {"policy_ids": [...], "installed": N}; 403 if the pack's category is not enabled
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/install/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
# Uninstall -> 204 (plugin 1.13+: allowed even when runs exist; history is kept under the policy name)
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/uninstall/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## YAML Import/Export

```bash
# Export single / all
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/1/export/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/export-all/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Import (creates, or updates the policy with the same name)
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/import/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d @my-policy.yaml
```

### YAML Format

```yaml
name: Leaf Intent Baseline
description: Standard intent validation for leaf switches
is_active: true
enable_config_engine: false
enable_graph_engine: false
trigger_on_branch_merge: false
trigger_on_cr_submit: true
schedule: "0 2 * * *"
sites: []            # scope by NetBox object *names* in YAML (regions, site_groups, sites, roles, platforms, device_tags)
roles: ["Leaf"]
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
```

Export writes only active rules and carries scope as object names (not IDs); import matches a policy by `name` and updates it in place.
