# API Patterns — NetBox Validation

All endpoints under `/api/plugins/validation/`. Auth: `Authorization: Bearer <token>`.

## Policy CRUD

```bash
# List policies
curl "$NETBOX_URL/api/plugins/validation/policies/" -H "Authorization: Bearer $NETBOX_TOKEN"

# Create a policy
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"name": "Leaf Baseline", "is_active": true, "enable_config_engine": true,
    "trigger_on_cr_submit": true, "schedule": "0 2 * * *",
    "scope_sites": [1, 2], "scope_roles": [3], "scope_tags": ["production"]}'

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

# Create a rule
curl -X POST "$NETBOX_URL/api/plugins/validation/rules/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "name": "Minimum Uplinks", "engine": "intent",
    "category": "redundancy", "check_name": "min_cabled_uplinks",
    "severity": "critical", "is_active": true, "parameters": {"min_uplinks": 4}}'

# Update a rule
curl -X PATCH "$NETBOX_URL/api/plugins/validation/rules/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"parameters": {"min_uplinks": 2}, "severity": "high"}'

# Delete a rule (plugin 1.13+: prior runs keep its name/outcome as "(deleted rule)")
curl -X DELETE "$NETBOX_URL/api/plugins/validation/rules/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## Run Lifecycle

```bash
# Create a run (status = pending, NOT executed)
curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "manual"}'

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
  -d '{"policy": 1, "trigger": "api", "target_sites": [9]}'

curl -X POST "$NETBOX_URL/api/plugins/validation/runs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"policy": 1, "trigger": "api", "target_devices": [101, 102, 103]}'

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
  -d '{"policy": 1, "trigger": "manual", "branch_id": 5}'
# Then execute as normal. Rules read from main; checks run in branch context.
```

## Results and Findings

```bash
# Results: filter by run, device, status, check_name
curl "$NETBOX_URL/api/plugins/validation/results/?run_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/results/?status=fail&device_id=10" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Findings: filter by severity, status, run
curl "$NETBOX_URL/api/plugins/validation/findings/?severity=critical&status=open" \
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

Remediating a finding means writing to core NetBox objects — on NetBox **4.7** selection custom fields read back as `{"value", "label"}` (write the raw value), services take `port_mappings`, and failed bulk writes report per-object errors by `index`. See [netbox-api-integration](../../netbox-api-integration/SKILL.md).

## Compliance Scores

```bash
curl "$NETBOX_URL/api/plugins/validation/compliance/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/?device_id=10&ordering=-measured_at" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/compliance/?policy_id=1" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

## Policy Packs

```bash
# List / details / install / uninstall
curl "$NETBOX_URL/api/plugins/validation/policy-packs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/install/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
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

# Import
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/import/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d @my-policy.yaml
```

### YAML Format

```yaml
name: Leaf Intent Baseline
description: Standard intent validation for leaf switches
enable_config_engine: false
enable_graph_engine: false
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
```
