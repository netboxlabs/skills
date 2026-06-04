---
name: netbox-automation-patterns
description: End-to-end automation patterns with NetBox — event-driven workflows (event rules, webhooks), infrastructure-as-code (Ansible, Terraform), and GitOps integration. Use when building, advising on, or troubleshooting NetBox automation pipelines.
license: Apache-2.0
---

# NetBox Automation Patterns

> **Your knowledge of NetBox automation tools may be outdated.** Ansible collection modules, Terraform provider resources, and event rule behavior change between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| NetBox Ansible collection | `https://github.com/netbox-community/ansible_modules` | Module list, parameters, examples |
| Ansible docs | `https://netboxlabs.com/docs/integrations/tool-integrations/netbox-ansible-collection/` | Integration guide |
| Terraform provider | `https://github.com/e-breuninger/terraform-provider-netbox` | Resources, data sources |
| Event rules docs | `https://netboxlabs.com/docs/netbox/features/event-rules/` | Webhook/script triggers |
| NetBox MCP server | If configured — verify event rules, webhooks exist | Validate automation config |

This skill covers how to integrate NetBox into automation workflows. It spans event-driven triggers, configuration management, infrastructure-as-code, and CI/CD pipelines. This is a cross-cutting skill — it ties together multiple open-source tools around NetBox as the source of truth.

**Load this skill when:**
- Building webhook-driven automation triggered by NetBox changes
- Configuring Ansible to use NetBox as dynamic inventory or state manager
- Managing NetBox resources via Terraform
- Designing GitOps pipelines with NetBox in the loop

**Out of scope:** NetBox core administration, plugin development, REST/GraphQL API basics (see `netbox-api-integration`).

---

## Quick Reference: Which Tool for Which Pattern?

| Goal | Tool | Pattern |
|------|------|---------|
| React to changes in NetBox | Event Rules + Webhooks | Push notification to external system |
| React to changes internally | Event Rules + Custom Scripts | Run logic inside NetBox |
| Define intended state in NetBox | Ansible modules (`netbox.netbox`) | Declarative, idempotent |
| Use NetBox as Ansible inventory | `nb_inventory` plugin | Dynamic inventory from NetBox data |
| Manage NetBox + cloud together | Terraform (`e-breuninger/netbox`) | IaC lifecycle, state tracking |
| Allocate next available IP/prefix | Terraform `available_*` resources | Auto-allocation from parent |
| NetBox → config generation → deploy | GitOps pipeline | CI/CD with NetBox as source of truth |

---

## Event-Driven Automation

NetBox's event rule system (introduced in 3.7) is the foundation for reactive automation. An **event rule** matches object changes to actions.

### Event Rule Essentials

- **Trigger scope**: Object type(s) + event type(s) (created, updated, deleted, job started/completed/failed/errored)
- **Conditions**: Optional JSON conditions to filter — e.g., only fire when `status.value == "active"`
- **Action types**: `webhook` (external HTTP call), `script` (run a custom script), `notification` (notify users)
- **Processing**: Asynchronous via Redis/RQ — the user request completes without waiting

### Key Guidelines

1. **Always scope with conditions** — Unscoped event rules fire on every matching change, creating noise and load. Use conditions to target specific statuses, roles, or tags.

2. **Events are async** — Don't assume immediate execution. The event is queued to Redis/RQ and processed by a worker. Design receivers to handle delays.

3. **Choose the right action type:**
   - External systems → webhook
   - Internal NetBox logic → custom script (see `netbox-custom-scripts`)
   - Human notification → notification group

4. **Test with the built-in receiver** before production:
   ```bash
   python netbox/manage.py webhook_receiver  # Listens on port 9000
   ```

5. **Coalescing behavior** — Multiple changes to the same object within one request are coalesced. Only the final state triggers the event. Delete events eagerly serialize data since the object won't exist later.

See [references/event-rules-and-webhooks.md](references/event-rules-and-webhooks.md) for payload format, HMAC signing, Jinja2 templating, and security considerations.

---

## Ansible Integration

The `netbox.netbox` Ansible collection (GPLv3) provides modules, dynamic inventory, and lookup plugins.

### Three Primary Use Cases

1. **State management** — Use modules (`netbox_device`, `netbox_ip_address`, etc.) to ensure NetBox objects match desired state. All modules are idempotent with `state: present/absent`.

2. **Dynamic inventory** — Use `nb_inventory` to generate Ansible inventory from NetBox. Group by device roles, sites, regions, tenants, tags, or platforms.

3. **Data lookup** — Use `nb_lookup` to query NetBox data within playbooks.

### Key Guidelines

1. **Set `config_context: False` in inventory** unless you need it — fetching config contexts adds significant overhead at scale.

2. **Use `query_filters`** to limit inventory scope server-side rather than filtering client-side.

3. **Scope tokens properly** — Read-only tokens for inventory/lookup, write tokens only for modules that create/modify objects.

4. **Version compatibility** — The collection supports the two most recent NetBox releases. Pin your collection version accordingly.

5. **Filter with `device_query_filters`** — e.g., `has_primary_ip: 'true'` to exclude devices without management IPs.

See [references/ansible-patterns.md](references/ansible-patterns.md) for module examples, inventory configuration, and lookup patterns.

---

## Terraform Integration

The `e-breuninger/netbox` Terraform provider manages NetBox resources as infrastructure-as-code.

### Key Guidelines

1. **Pin provider version to match NetBox version** — NetBox makes breaking API changes in minor releases. Check the provider's compatibility matrix.

2. **Understand `available_*` resource lifecycle** — `netbox_available_ip_address` and `netbox_available_prefix` allocate on create and cannot be "updated." They have unique lifecycle behavior compared to regular resources.

3. **Plan for state drift** — If someone modifies NetBox outside Terraform, the next `terraform plan` shows drift. Decide on a drift remediation strategy.

4. **Coverage varies by area** — The provider has strongest coverage for IPAM and virtualization. Other models may have incomplete resource support.

5. **Use data sources for read-only lookups** — Most resources have corresponding data sources for referencing existing NetBox objects without managing them.

See [references/terraform-patterns.md](references/terraform-patterns.md) for provider setup, resource examples, and the `available_*` allocation pattern.

---

## GitOps Workflows

NetBox integrates into GitOps pipelines as either the **source of truth** or a **state mirror**.

### Common Architectures

| Pattern | Flow | Best For |
|---------|------|----------|
| Webhook-driven | NetBox change → webhook → CI/CD → deploy | Real-time reactions |
| Polling/export | Cron → export NetBox data → Git commit → CI/CD | Batch config generation |
| Ansible pull | Ansible + nb_inventory → generate configs → push | Playbook-driven workflows |
| Terraform unified | Terraform manages NetBox + infrastructure together | Cloud + NetBox in sync |

### Key Guidelines

1. **Decide on a single source of truth** — Either NetBox prescribes state and Git/automation enforces it, or Git is the source and NetBox mirrors it. Don't have two masters.

2. **Use config contexts for template data** — Config contexts provide per-device, per-role, or per-site configuration data that Jinja2 templates consume during config generation.

3. **Webhook → CI/CD for real-time pipelines** — Combine event rules with webhooks to trigger GitHub Actions, GitLab CI, or Jenkins on NetBox changes.

4. **Use tags as automation hints** — Tag interfaces, devices, or prefixes to signal automation intent (e.g., tag "OSPF" on an interface to include it in OSPF config generation).

See [references/gitops-workflows.md](references/gitops-workflows.md) for pipeline architecture details and examples.

---

## Cross-Cutting Concerns

These apply across all automation approaches:

- **Token management**: Use scoped tokens with minimal permissions. Prefer v2 tokens (NetBox 4.5+) with `Bearer` auth. See [netbox-api-integration](../netbox-api-integration/SKILL.md) for token format details.

- **Rate limiting**: Large automation runs (bulk Ansible plays, Terraform applies) should implement backoff to avoid overwhelming the NetBox API.

- **Pagination**: All tools (pynetbox, Terraform provider, Ansible collection) handle pagination internally, but be aware of it when writing custom integrations. NetBox **4.6** adds cursor-based `start` pagination (an efficient alternative to deep `offset` scans) — see [netbox-api-integration](../netbox-api-integration/SKILL.md).

- **Idempotency**: Ansible modules are idempotent by design. Terraform is declarative. Webhooks are fire-and-forget — implement idempotency on the receiver side.

- **Testing**: Use NetBox's official Docker image for CI/CD testing environments. Spin up a disposable instance for integration tests.

---

## References

| Document | When to Load |
|----------|-------------|
| [references/event-rules-and-webhooks.md](references/event-rules-and-webhooks.md) | Building webhook integrations, configuring event rules, debugging webhook delivery |
| [references/ansible-patterns.md](references/ansible-patterns.md) | Writing Ansible playbooks that manage or query NetBox |
| [references/terraform-patterns.md](references/terraform-patterns.md) | Managing NetBox resources with Terraform |
| [references/gitops-workflows.md](references/gitops-workflows.md) | Designing CI/CD pipelines with NetBox in the loop |
