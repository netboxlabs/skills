# Branch-Aware Models

## What Gets Branched

Most core NetBox operational data models support branching. The exact set depends on plugin configuration and NetBox version — use the [discovery endpoint](#discovery-endpoint) for the authoritative list.

In practice, **most operational data models** are branched:
- **DCIM**: sites, devices, interfaces, cables, racks, etc.
- **IPAM**: prefixes, IP addresses, VLANs, VRFs, etc.
- **Circuits**: circuits, providers, circuit terminations
- **Tenancy**: tenants, tenant groups, contacts
- **Virtualization**: clusters, VMs, VM interfaces
- **VPN**: tunnels, IKE/IPSec profiles
- **Wireless**: wireless LANs, wireless links

## What Is NOT Branched (Exempt)

These models are **global** — changes are immediate and affect all branches:

| Category | Models |
|----------|--------|
| Core (all) | Jobs, data sources, data files, etc. |
| Config objects | Custom fields, custom field choice sets |
| Automation | Webhooks, event rules, custom links, export templates |
| Admin | Notification groups, saved filters |
| Branching plugin | All `netbox_branching.*` models (branches, events, diffs) |
| Changes plugin | All `netbox_changes.*` models (change requests, approvals) |
| Additional | Any models the admin adds to `exempt_models` in plugin config |

**Key implication:** If you create a custom field while a branch is active, it applies globally to main and all branches immediately.

## Identifying Branchable Models

> **Tip:** `GET /api/plugins/branching/branchable-models/` lists all branchable models on your install — the authoritative source. It ships across the supported 4.4.1+ / plugin 1.0.x range (some older 0.8.x builds returned 404). Prefer it over the heuristics below when you need certainty.

**In practice, the rule is simple:**

- **Most core operational data models are branched** — anything you'd find under DCIM, IPAM, Circuits, Tenancy, Virtualization, VPN, and Wireless.
- **Infrastructure/config models are NOT branched** — custom fields, webhooks, event rules, export templates, data sources, jobs, and all `core.*` models.
- **Plugin models are NOT branched** — `netbox_branching.*`, `netbox_changes.*`, and other plugin models are exempt by default.
- **Admin-configured exemptions** — admins can add models to `exempt_models` in plugin config.

If you need to confirm whether a specific model is branchable, try creating/modifying an object of that type within a branch context. If the model isn't branched, the change will apply to main directly.

## Practical Guidance

- **Assume core operational models are branched** unless they fall into the exempt categories above.
- **Global config changes** (custom fields, webhooks) don't need branches — they take effect immediately.
- **M2M relationships** (e.g., tag assignments) are branched via their through tables.
- **Cable paths** are branched — topology changes in a branch don't affect main's path calculations.
