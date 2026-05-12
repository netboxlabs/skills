# NetBox Skills — Agent Quick Reference

For non-Claude agents: load the hub skill first, then drill into specialists.

## Getting Started

1. Load `skills/netbox/SKILL.md` — the hub skill with decision trees
2. Based on the task, load the appropriate specialized skill
3. Load reference files only when you need specific details

## Skills

| Skill | Path | Use When |
|-------|------|----------|
| netbox | `skills/netbox/SKILL.md` | Starting point — routes to specialists |
| netbox-api-integration | `skills/netbox-api-integration/SKILL.md` | Querying or writing data via REST/GraphQL |
| netbox-data-modeling | `skills/netbox-data-modeling/SKILL.md` | Designing network data models |
| netbox-custom-objects | `skills/netbox-custom-objects/SKILL.md` | Creating custom object types |
| netbox-plugin-development | `skills/netbox-plugin-development/SKILL.md` | Building NetBox plugins |
| netbox-custom-scripts | `skills/netbox-custom-scripts/SKILL.md` | Writing scripts that run inside NetBox |
| netbox-config-templates | `skills/netbox-config-templates/SKILL.md` | Generating device configs with Jinja2 |
| netbox-administration | `skills/netbox-administration/SKILL.md` | Server config, auth, performance |
| netbox-branching | `skills/netbox-branching/SKILL.md` | Change isolation with branches |
| netbox-changes | `skills/netbox-changes/SKILL.md` | Change request workflows |
| netbox-diode | `skills/netbox-diode/SKILL.md` | Bulk data ingestion via gRPC |
| netbox-discovery | `skills/netbox-discovery/SKILL.md` | Network discovery with Orb Agent |
| netbox-validation | `skills/netbox-validation/SKILL.md` | Validation policies, runs, findings, compliance |
| netbox-assurance | `skills/netbox-assurance/SKILL.md` | Drift detection and remediation |
| netbox-automation-patterns | `skills/netbox-automation-patterns/SKILL.md` | Webhooks, Ansible, Terraform, GitOps |
| netbox-migration | `skills/netbox-migration/SKILL.md` | Data migration into NetBox |
| netbox-review-integration | `skills/netbox-review-integration/SKILL.md` | Reviewing integration code quality |
| netbox-review-datamodel | `skills/netbox-review-datamodel/SKILL.md` | Auditing data model design |

## Progressive Disclosure

Skills use a layered approach to minimize context usage:

- **SKILL.md** — Core instructions, always load first (< 500 lines each)
- **references/** — Detailed docs, load only when you need specifics
- **scripts/** — Executable code, run when needed

## Commands

| Command | Path | Purpose |
|---------|------|---------|
| setup-mcp | `commands/setup-mcp.md` | Configure NetBox MCP server |
| build-plugin | `commands/build-plugin.md` | Scaffold a new plugin |
