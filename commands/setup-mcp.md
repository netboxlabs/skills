---
name: setup-mcp
description: Guide the user through setting up a NetBox MCP server for AI agent access.
---

# Setup NetBox MCP Server

Help the user connect their AI agent to NetBox via MCP (Model Context Protocol).

## Step 1: Determine Deployment Type

Ask the user:
- **NetBox Community** (open-source, self-hosted)?
- **NetBox Cloud or Enterprise** (commercial platform)?

---

## Community Edition

Use `netbox-mcp-server` — the open-source, read-only MCP server for NetBox.

**Repository:** https://github.com/netboxlabs/netbox-mcp-server

### Create a token

Create a **read-only** API token for the server (Settings > API Tokens in the NetBox UI). The server never writes, so the token needs only view permissions on the models you want to expose. Use an `nbt_`-prefixed v2 token on NetBox 4.5+.

### Install with Claude Code

Pass the URL and token as environment variables:

```bash
claude mcp add --transport stdio netbox \
  --env NETBOX_URL=https://YOUR_NETBOX_URL/ \
  --env NETBOX_TOKEN=YOUR_API_TOKEN \
  -- uvx netbox-mcp-server
```

The server's command-line flags are `--netbox-url` and `--netbox-token`; `--url` and `--token` fail with `unrecognized arguments`. Prefer the environment variables over the flags, which put the token in the process arguments. Either way the token is stored in plain text in the agent's MCP configuration — another reason to keep it read-only.

### Install with other agents

Add to your MCP configuration (e.g., `.cursor/mcp.json`, `cline_mcp_settings.json`):

```json
{
  "mcpServers": {
    "netbox": {
      "command": "uvx",
      "args": ["netbox-mcp-server"],
      "env": {
        "NETBOX_URL": "https://YOUR_NETBOX_URL/",
        "NETBOX_TOKEN": "YOUR_API_TOKEN"
      }
    }
  }
}
```

### Requirements

- Python 3.11+ with `uv` installed
- A read-only NetBox API token (see above)

### Writes

The server is read-only. Make changes through the REST API ([netbox-api-integration](../skills/netbox-api-integration/SKILL.md)) or Diode ([netbox-diode](../skills/netbox-diode/SKILL.md)), with a separate write-enabled token.

### Verify

After adding, test with:
- Ask your agent: "List all sites in NetBox"
- The agent should use the MCP tools to query your instance

---

## Cloud / Enterprise Edition

Use the NetBox Labs Platform MCP Server — the commercial MCP server with full CRUD access, branching support, and enterprise auth.

**Status:** Public Preview. NetBox Labs support enables it on your instance.

- **Endpoint:** `https://<instance>.cloud.netboxapp.com/mcp` (streamable HTTP; there is no stdio mode)
- **Auth:** `Authorization: Bearer nbt_<token>` — a v2 token; its NetBox permissions apply to every call
- **Read-only connections:** append `?read_only=true` to the endpoint URL

Connection options, Code Mode vs Discrete mode, and usage patterns: [netboxlabs-platform-mcp](../skills/netboxlabs-platform-mcp/SKILL.md).

### What it provides

- Full read/write access to NetBox
- Branch-aware operations (create, stage, merge)
- Enterprise SSO authentication
- Audit logging for all agent actions

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Connection refused | Verify NetBox URL is reachable from your machine |
| 403 Forbidden | Check token permissions — ensure it has read access to required models |
| MCP server not found | Ensure `uv` is installed: `pip install uv` or `brew install uv` |
| `unrecognized arguments: --url` | Use the `NETBOX_URL` / `NETBOX_TOKEN` environment variables (or `--netbox-url` / `--netbox-token`) |
| Token format | Use `nbt_` prefixed tokens on NetBox 4.5+ (Settings > API Tokens) |
