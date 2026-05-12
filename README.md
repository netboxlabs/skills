# NetBox Labs Skills

[Agent Skills](https://agentskills.io) for the NetBox ecosystem — structured knowledge that helps AI agents work effectively with NetBox, its APIs, and platform products.

## Installing

### Claude Code

```bash
/plugin marketplace add netboxlabs/skills
```

### Cursor

Add to your Cursor settings:

```json
{
  "skills": ["netboxlabs/skills"]
}
```

### npx

```bash
npx skills add netboxlabs/skills
```

### Clone / Copy

| Method | Command |
|--------|---------|
| Clone | `git clone https://github.com/netboxlabs/skills.git` |
| Copy a skill | Copy any `skills/<name>/SKILL.md` + `references/` into your project |

## Commands

| Command | Description |
|---------|-------------|
| `setup-mcp` | Configure a NetBox MCP server for AI agent access |
| `build-plugin` | Scaffold a new NetBox plugin with standard project structure |

## Skills

| Skill | Description |
|-------|-------------|
| [netbox](skills/netbox/) | Hub skill — routes to the right specialist for any NetBox task |
| [netbox-api-integration](skills/netbox-api-integration/) | REST & GraphQL API patterns, pynetbox, authentication |
| [netbox-data-modeling](skills/netbox-data-modeling/) | Model networks — hierarchy, IPAM, tenancy, custom fields |
| [netbox-custom-objects](skills/netbox-custom-objects/) | No-code data model extensibility — custom types, fields, relationships |
| [netbox-plugin-development](skills/netbox-plugin-development/) | Building plugins — models, views, APIs, migrations, testing |
| [netbox-custom-scripts](skills/netbox-custom-scripts/) | Custom scripts & reports — Script class, jobs, ORM access |
| [netbox-config-templates](skills/netbox-config-templates/) | Config generation — Jinja2, context variables, platform patterns |
| [netbox-administration](skills/netbox-administration/) | Server admin — config, auth, permissions, performance |
| [netbox-branching](skills/netbox-branching/) | Branch lifecycle, schema isolation, CR workflows |
| [netbox-changes](skills/netbox-changes/) | Change management, policies, approval workflows |
| [netbox-diode](skills/netbox-diode/) | Data ingestion — SDKs, entity mapping, reconciler |
| [netbox-discovery](skills/netbox-discovery/) | Orb Agent — config, backends, policies, secrets |
| [netbox-validation](skills/netbox-validation/) | Validation policies, compliance checks, findings, pre-change safety |
| [netbox-assurance](skills/netbox-assurance/) | Drift detection — intended vs actual, remediation |
| [netbox-automation-patterns](skills/netbox-automation-patterns/) | Webhooks, Ansible, Terraform, GitOps |
| [netbox-migration](skills/netbox-migration/) | Migrating from spreadsheets, CMDBs, other tools |
| [netbox-review-integration](skills/netbox-review-integration/) | Review integration code for correctness and performance |
| [netbox-review-datamodel](skills/netbox-review-datamodel/) | Audit data model design for best practices |

## MCP Servers

| Server | Access | Status | Link |
|--------|--------|--------|------|
| `netbox-mcp-server` | Read-only (OSS) | Available | [GitHub](https://github.com/netboxlabs/netbox-mcp-server) |
| NetBox Labs Platform MCP Server | Full CRUD (Commercial) | Coming soon | — |

Run `/setup-mcp` to configure MCP server access for your agent.

## Resources

- [NetBox Documentation](https://netboxlabs.com/docs/netbox/)
- [NetBox GitHub](https://github.com/netbox-community/netbox)
- [NetBox Labs](https://netboxlabs.com)
- [Agent Skills Spec](https://agentskills.io)
- [pynetbox SDK](https://github.com/netbox-community/pynetbox)

## License

[Apache 2.0](LICENSE)
