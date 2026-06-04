---
name: netbox
description: >
  Hub skill for the NetBox ecosystem. Routes to specialized skills for data modeling,
  API integration, plugin development, change management, discovery, assurance,
  automation, and migration. Load this skill first to find the right specialist.
license: Apache-2.0
---

# NetBox Hub Skill

This is the entry point for all NetBox-related tasks. Use the decision trees below to find the right specialized skill, then load it for detailed guidance.

## Decision Trees

### What are you trying to do?

**Model your network?**
- Define sites, devices, circuits, IPAM → [netbox-data-modeling](../netbox-data-modeling/SKILL.md)
- Create new object types without code → [netbox-custom-objects](../netbox-custom-objects/SKILL.md)

**Build an integration?**
- Query or write data via REST/GraphQL → [netbox-api-integration](../netbox-api-integration/SKILL.md)
- Bulk ingest data via gRPC (Diode SDK) → [netbox-diode](../netbox-diode/SKILL.md)
- Automate with webhooks, Ansible, Terraform → [netbox-automation-patterns](../netbox-automation-patterns/SKILL.md)
- Drive NetBox from an AI agent via the Platform MCP server → [netboxlabs-platform-mcp](../netboxlabs-platform-mcp/SKILL.md)

**Extend NetBox?**
- Build a plugin (models, views, APIs) → [netbox-plugin-development](../netbox-plugin-development/SKILL.md)
- Write scripts that run inside NetBox → [netbox-custom-scripts](../netbox-custom-scripts/SKILL.md)
- Generate device configs with Jinja2 → [netbox-config-templates](../netbox-config-templates/SKILL.md)

**Manage changes safely?**
- Stage changes in isolated branches → [netbox-branching](../netbox-branching/SKILL.md)
- Track change requests and approvals → [netbox-changes](../netbox-changes/SKILL.md)

**Validate your network?**
- Check compliance against policies → [netbox-validation](../netbox-validation/SKILL.md)
- Detect drift between intended and actual → [netbox-assurance](../netbox-assurance/SKILL.md)

**Discover or audit your network?**
- Auto-discover infrastructure with Orb Agent → [netbox-discovery](../netbox-discovery/SKILL.md)
- Detect drift between intended and actual state → [netbox-assurance](../netbox-assurance/SKILL.md)

**Procure or spare equipment?**
- Track procurement (BOMs, POs, shipments) and spares → [netbox-asset-lifecycle](../netbox-asset-lifecycle/SKILL.md)

**Look up hardware specs or lifecycle data?**
- Consume device-type metadata and enrichment (EOL, thermal, protocols) → [netbox-ndx](../netbox-ndx/SKILL.md)

**Operate or migrate?**
- Configure, secure, or tune a NetBox server → [netbox-administration](../netbox-administration/SKILL.md)
- Migrate data from spreadsheets or legacy tools → [netbox-migration](../netbox-migration/SKILL.md)

**Review or audit existing work?**
- Review integration code (pagination, auth, performance) → [netbox-review-integration](../netbox-review-integration/SKILL.md)
- Audit data model design (hierarchy, IPAM, naming) → [netbox-review-datamodel](../netbox-review-datamodel/SKILL.md)

---

## Skill Index

| Skill | Category | Description |
|-------|----------|-------------|
| [netbox-data-modeling](../netbox-data-modeling/SKILL.md) | Core | Model networks — hierarchy, IPAM, tenancy, custom fields |
| [netbox-custom-objects](../netbox-custom-objects/SKILL.md) | Core | No-code data model extensibility — custom types, fields, relationships |
| [netbox-api-integration](../netbox-api-integration/SKILL.md) | Core | REST & GraphQL API patterns, pynetbox, authentication |
| [netbox-plugin-development](../netbox-plugin-development/SKILL.md) | Core | Building plugins — models, views, APIs, migrations, testing |
| [netbox-custom-scripts](../netbox-custom-scripts/SKILL.md) | Core | Custom scripts & reports — Script class, jobs, ORM access |
| [netbox-config-templates](../netbox-config-templates/SKILL.md) | Core | Config generation — Jinja2, context variables, platform patterns |
| [netbox-administration](../netbox-administration/SKILL.md) | Core | Server admin — config, auth, permissions, performance |
| [netbox-branching](../netbox-branching/SKILL.md) | Platform | Branch lifecycle, schema isolation, CR workflows |
| [netbox-changes](../netbox-changes/SKILL.md) | Platform | Change management, policies, approval workflows |
| [netbox-diode](../netbox-diode/SKILL.md) | Platform | Data ingestion — SDKs, entity mapping, reconciler |
| [netbox-discovery](../netbox-discovery/SKILL.md) | Platform | Orb Agent — config, backends, policies, secrets |
| [netbox-validation](../netbox-validation/SKILL.md) | Platform | Validation policies, compliance checks, findings, pre-change safety |
| [netbox-assurance](../netbox-assurance/SKILL.md) | Platform | Drift detection — intended vs actual, remediation |
| [netbox-asset-lifecycle](../netbox-asset-lifecycle/SKILL.md) | Platform | Procurement lifecycle — BOMs, POs, shipments, receiving, spares |
| [netbox-ndx](../netbox-ndx/SKILL.md) | Platform | Data Exchange — device-type catalog + enrichment (lifecycle, thermal, protocols) |
| [netboxlabs-platform-mcp](../netboxlabs-platform-mcp/SKILL.md) | Platform | Platform MCP server — code mode & discrete mode for agents |
| [netbox-automation-patterns](../netbox-automation-patterns/SKILL.md) | Cross-cutting | Webhooks, Ansible, Terraform, GitOps |
| [netbox-migration](../netbox-migration/SKILL.md) | Cross-cutting | Migrating from spreadsheets, CMDBs, other tools |
| [netbox-review-integration](../netbox-review-integration/SKILL.md) | Review | Audit integration code for correctness and performance |
| [netbox-review-datamodel](../netbox-review-datamodel/SKILL.md) | Review | Audit data model design for best practices |

---

## Common Patterns

### Progressive Disclosure

Each skill follows a layered structure:
1. **SKILL.md** — Core instructions (< 500 lines). Load this first.
2. **references/** — Detailed docs loaded on demand when you need specifics.
3. **scripts/** — Executable code (where applicable).

Load only what you need. Start with the hub, pick a specialized skill, then drill into references as required.

### Multi-Skill Tasks

Some tasks span multiple skills. Common combinations:

| Task | Skills |
|------|--------|
| Build a discovery integration | `netbox-diode` + `netbox-discovery` |
| Plugin with change management | `netbox-plugin-development` + `netbox-branching` |
| Migrate then validate | `netbox-migration` + `netbox-assurance` |
| Automate config generation | `netbox-config-templates` + `netbox-automation-patterns` |
| Model + populate data | `netbox-data-modeling` + `netbox-api-integration` |
| Review a full integration | `netbox-review-integration` + `netbox-review-datamodel` |
| Pre-change validation | `netbox-validation` + `netbox-branching` + `netbox-changes` |
| Procure planned equipment | `netbox-data-modeling` + `netbox-asset-lifecycle` |
| EOL exposure across inventory | `netbox-ndx` + `netbox-api-integration` |
| Agent-driven NetBox automation | `netboxlabs-platform-mcp` + `netbox-api-integration` |

---

## Principles

When working on any NetBox task:

- **Prefer retrieval over pre-training.** NetBox APIs, configuration, and plugin behavior change between releases. Always verify against current docs or a live instance before generating code or making assertions.
- **Use MCP when available.** If a NetBox MCP server is configured, query the live instance to verify object schemas, check existing data, and validate your work.
- **Respect dependency order.** NetBox objects have parent-child relationships. Create parents before children (or use Diode to skip ordering).
- **Be explicit about versions.** When giving advice that depends on a NetBox version (v2 tokens, cursor pagination, custom objects), state the minimum version clearly.
- **Correctness over cleverness.** A working integration with clear code is better than a compact one that's fragile.

---

## References

- [Product Catalog](references/product-catalog.md) — All NetBox Labs products and their interfaces
- [Version Matrix](references/netbox-version-matrix.md) — Version-dependent features across the platform
