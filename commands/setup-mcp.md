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

### Install with Claude Code

```bash
claude mcp add netbox-mcp-server -- uvx netbox-mcp-server --url https://YOUR_NETBOX_URL --token YOUR_API_TOKEN
```

### Install with other agents

Add to your MCP configuration (e.g., `.cursor/mcp.json`, `cline_mcp_settings.json`):

```json
{
  "mcpServers": {
    "netbox-mcp-server": {
      "command": "uvx",
      "args": ["netbox-mcp-server", "--url", "https://YOUR_NETBOX_URL", "--token", "YOUR_API_TOKEN"]
    }
  }
}
```

### Requirements

- Python 3.11+ with `uv` installed
- A NetBox API token (Settings > API Tokens in NetBox UI)
- Use a `nbt_`-prefixed v2 token on NetBox 4.5+

### Verify

After adding, test with:
- Ask your agent: "List all sites in NetBox"
- The agent should use the MCP tools to query your instance

---

## Cloud / Enterprise Edition

Use the NetBox Labs Platform MCP Server — the commercial MCP server with full CRUD access, branching support, and enterprise auth.

**Status:** Coming soon. Contact NetBox Labs for early access.

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
| Token format | Use `nbt_` prefixed tokens on NetBox 4.5+ (Settings > API Tokens) |
