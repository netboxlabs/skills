# GitOps Workflows with NetBox

---

## Architecture Patterns

### Pattern 1: Webhook-Driven Pipeline

```
NetBox Change → Event Rule → Webhook → CI/CD → Deploy
```

**How it works:**
1. Engineer updates a device/prefix/VLAN in NetBox
2. Event rule fires a webhook to a CI/CD endpoint (GitHub Actions, GitLab CI, Jenkins)
3. CI/CD pipeline generates configs from NetBox data and deploys

**Best for:** Real-time reactions to changes. Low latency between intent and deployment.

**Webhook → GitHub Actions example:**
- Configure a webhook pointing to a GitHub repository dispatch endpoint
- Use a body template to include relevant data
- GitHub Actions workflow triggers on `repository_dispatch` event

### Pattern 2: Polling/Export to Git

```
Cron → Export NetBox Data → Commit to Git → CI/CD Detects Change → Deploy
```

**How it works:**
1. A scheduled job (cron, Ansible, custom script) exports NetBox data to YAML/JSON files
2. Files are committed to a Git repository
3. CI/CD detects the commit and runs the deployment pipeline

**Best for:** Batch operations, audit trail via Git history, environments where webhooks aren't feasible.

### Pattern 3: Ansible Pull from NetBox

```
Trigger → Ansible (nb_inventory + nb_lookup) → Generate Configs → Push to Devices
```

**How it works:**
1. Ansible uses `nb_inventory` for dynamic inventory and `nb_lookup` for config data
2. Jinja2 templates generate device configurations using NetBox data
3. Ansible pushes configs to devices (via NAPALM, netmiko, etc.)

**Best for:** Teams already using Ansible. Can be triggered by cron, CI/CD, or webhooks.

See [ansible-patterns.md](ansible-patterns.md) for inventory and module details.

### Pattern 4: Terraform Unified Management

```
Terraform → NetBox (IP/device records) + Cloud Provider (actual infra)
```

**How it works:**
1. Single Terraform codebase manages both NetBox records and infrastructure resources
2. NetBox allocations (IPs, prefixes) feed into cloud provider resources
3. State stays in sync via Terraform's state file

**Best for:** Cloud-centric environments where NetBox tracks IP allocations consumed by cloud resources.

See [terraform-patterns.md](terraform-patterns.md) for the combined pattern example.

---

## Source of Truth Decision

The most critical design decision: **is NetBox the prescriptive source of truth, or a reflective mirror?**

| Approach | NetBox Role | Git Role | When to Use |
|----------|-------------|----------|-------------|
| NetBox-prescriptive | Defines intended state | Stores generated configs | Network teams managing physical/virtual infra directly |
| Git-prescriptive | Mirrors actual state | Defines intended state | Teams with strong GitOps culture, NetBox populated from discovered state |
| Hybrid | Source for IPAM/DCIM | Source for config templates | Most common — NetBox owns data models, Git owns config logic |

**Don't have two masters.** If both NetBox and Git can independently change the same data, you'll have constant conflicts.

---

## Config Context for Template Data

Config contexts are a powerful mechanism for attaching structured data to devices, roles, sites, or other objects. Use them to provide per-scope template variables:

```yaml
# Config context assigned to role "leaf-switch"
bgp:
  asn: 65001
  neighbors:
    - peer: 10.0.0.1
      remote_asn: 65000
ntp_servers:
  - 10.0.0.253
  - 10.0.0.254
```

Templates consume this data during config generation:

```jinja2
router bgp {{ bgp.asn }}
{% for neighbor in bgp.neighbors %}
  neighbor {{ neighbor.peer }} remote-as {{ neighbor.remote_asn }}
{% endfor %}
```

Config contexts merge hierarchically (region → site → role → device), with more specific contexts taking precedence.

---

## Tags as Automation Signals

Use NetBox tags to signal automation intent without adding custom fields:

| Tag | Meaning | Automation Action |
|-----|---------|------------------|
| `ospf` | Interface participates in OSPF | Include in OSPF config block |
| `monitored` | Device should be in monitoring | Enroll in monitoring system |
| `decommission` | Device pending removal | Trigger decommission workflow |
| `terraform-managed` | Managed by Terraform | Skip in Ansible runs |

Event rules can use conditions to trigger only when specific tags are present.

---

## Pipeline Validation

Before deploying generated configs, validate in CI/CD:

1. **Syntax check** — Lint generated configs (e.g., `batfish`, vendor-specific linters)
2. **Policy check** — Verify compliance rules (no overlapping subnets, required tags present)
3. **Dry run** — Test against a lab or simulation environment
4. **Approval gate** — Require human approval for production changes
5. **Post-deploy verification** — Confirm actual state matches intended state

---

## Testing with NetBox Docker

Use NetBox's official Docker image in CI/CD for integration testing:

```yaml
# GitHub Actions example
services:
  netbox:
    image: netboxcommunity/netbox:v4.6-3.4.0   # pin to the NetBox version you test against; ':latest' silently drifts (e.g. into 4.6/Django 6.0)
    ports:
      - 8000:8080
    env:
      SUPERUSER_API_TOKEN: "0123456789abcdef0123456789abcdef01234567"
```

Seed test data via Ansible modules or direct API calls at the start of each pipeline run.
