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

> **Tip:** `GET /api/plugins/branching/branchable-models/` lists all branchable models on your install — the authoritative source, available on both supported lines (1.1.x for NetBox 4.4.1–4.6, 1.2.x for 4.7). Prefer it over the heuristics below when you need certainty.

**In practice, the rule is simple:**

- **Any change-logged model is branched** — everything under DCIM, IPAM, Circuits, Tenancy, Virtualization, VPN, Wireless, plus NetBox 4.7's new cooling and module-bay-type models, and **any plugin model that inherits `NetBoxModel`/`ChangeLoggingMixin`** (branching support is automatic; a plugin must opt *out* via `exempt_models`).
- **Infrastructure/config models are NOT branched** — custom fields, choice sets, custom links, webhooks, event rules, export templates, saved filters, notification groups, and all `core.*` models (data sources, jobs).
- **The branching and Changes plugins' own models are NOT branched** — `netbox_branching.*` and `netbox_changes.*` are built-in exemptions.
- **Models without change logging are NOT branched** — except a fixed set of through/cache tables the plugin replicates for correctness (tag assignments, cable paths, port mappings, contact group membership, cached search values). Multi-table inheritance is unsupported and breaks provisioning.
- **Admin-configured exemptions** — admins can add models to `exempt_models` (`'plugin.model'` or `'plugin.*'`); never exempt a model that has a relationship to a branched model.

> **Custom Objects plugin:** custom-object *instance* writes in a branch are rejected on 0.5.x and supported (version-gated) from 0.6.0. Type/field definitions always apply to main. See [netbox-custom-objects](../../netbox-custom-objects/SKILL.md).

> **NetBox 4.7 hierarchies:** region, site group, location, device role, platform, tenant group, contact group, wireless LAN group, module bay, and inventory item paths are `ltree` columns maintained by DB triggers. Branching 1.2.x replicates those triggers into each branch schema, so parent moves/renames cascade correctly inside the branch. Branching 1.1.x does not load on 4.7.

If you need to confirm whether a specific model is branchable, try creating/modifying an object of that type within a branch context. If the model isn't branched, the change will apply to main directly.

## Practical Guidance

- **Assume core operational models are branched** unless they fall into the exempt categories above.
- **Global config changes** (custom fields, webhooks) don't need branches — they take effect immediately.
- **M2M relationships** (e.g., tag assignments) are branched via their through tables.
- **Cable paths** are branched — topology changes in a branch don't affect main's path calculations.
