# Terraform Integration Patterns

Provider: `e-breuninger/netbox` — open source, installed via Terraform registry.

---

## Provider Setup

```hcl
terraform {
  required_providers {
    netbox = {
      source  = "e-breuninger/netbox"
      version = "~> 5.0"  # Pin to match your NetBox version
    }
  }
}

provider "netbox" {
  server_url = "https://netbox.example.com"
  api_token  = var.netbox_token
}
```

**Always pin the provider version.** NetBox makes breaking API changes in minor releases, and the provider version must match. Check the provider's compatibility matrix:

| Provider Version | NetBox Version (per provider README, checked 2026-09-30) |
|-----------------|---------------|
| v5.6.1 – v5.8.0 | 4.3.0 – 4.6.5 |
| v5.0.0 – v5.6.0 | 4.3.0 – 4.4.10 |
| v6.0.0-rc.x | 4.6.8 – 4.6.10 (full rewrite, see below) |

> **No provider release declares NetBox 4.7 support yet.** The provider probes the NetBox version at init and emits a non-blocking warning when unsupported. 4.7 changes most likely to bite the v5.x provider: selection custom fields now read back as `{value,label}` objects (perpetual diff or type error on `custom_fields`), `config_context` always present on devices/VMs, and `netbox_service` `protocol`/`ports` returning `null` for multi-protocol services. Test against 4.7 in staging before pinning, and re-check the README matrix.
>
> **v6.0.0** (release candidates since 2026-09-26) is a generated rewrite on the Terraform Plugin Framework with breaking changes: `netbox_interface` → `netbox_virtual_machine_interface`, `netbox_device_primary_ip` merged into `netbox_primary_ip`, tags referenced by slug, data-source `filter` blocks → `filters`, custom-field lookups → `custom_field_filters` / `cf_<name>`. Stay on `~> 5.0` for existing state until you plan the migration.

---

## Resource Patterns

### Standard Resources

```hcl
resource "netbox_site" "dc1" {
  name   = "DC1"
  slug   = "dc1"
  status = "active"
}

resource "netbox_device_role" "leaf" {
  name      = "Leaf Switch"
  slug      = "leaf-switch"
  color_hex = "00ff00"
}

resource "netbox_vlan" "mgmt" {
  name = "Management"
  vid  = 100
}
```

### Available Resource Allocation

The `available_*` resources are special — they allocate the **next available** resource from a parent:

```hcl
# Allocate next available /24 from a supernet
resource "netbox_available_prefix" "server_net" {
  parent_prefix_id = netbox_prefix.supernet.id
  prefix_length    = 24
  status           = "active"
  description      = "Server network"
}

# Allocate next available IP from a prefix
resource "netbox_available_ip_address" "server1" {
  prefix_id   = netbox_prefix.server_net.id
  dns_name    = "server1.example.com"
  status      = "active"
}
```

**Lifecycle behavior:**
- Allocated on `terraform apply` (create)
- Cannot be updated in place — changes force replacement
- Destroying the resource frees the allocation in NetBox
- If no IP/prefix is available, the apply fails

### Data Sources (Read-Only Lookups)

Reference existing NetBox objects without managing them:

```hcl
data "netbox_site" "existing" {
  name = "DC1"
}

resource "netbox_rack" "rack1" {
  name    = "Rack 1"
  site_id = data.netbox_site.existing.id
}
```

---

## Key Resource Coverage

| Category | Resources | Data Sources |
|----------|-----------|-------------|
| DCIM | `netbox_device`, `netbox_site`, `netbox_rack`, `netbox_device_interface`, `netbox_platform`, `netbox_manufacturer`, `netbox_cable` | Yes |
| IPAM | `netbox_ip_address`, `netbox_prefix`, `netbox_vlan`, `netbox_vrf`, `netbox_available_ip_address`, `netbox_available_prefix` | Yes |
| Virtualization | `netbox_cluster`, `netbox_virtual_machine`, `netbox_interface` (VM interface; `netbox_virtual_machine_interface` in v6) | Yes |
| Tenancy | `netbox_tenant`, `netbox_contact`, `netbox_contact_assignment` | Yes |
| Extras | `netbox_tag`, `netbox_custom_field`, `netbox_config_context`, `netbox_webhook`, `netbox_event_rule` | Partial |

---

## State Drift

Terraform expects to be the sole manager of resources it tracks. When someone modifies a NetBox object outside Terraform:

- `terraform plan` shows the drift as a proposed change
- `terraform apply` reverts the external change to match the Terraform config

**Strategies:**
1. **Prevent drift**: Use NetBox permissions to restrict manual edits on Terraform-managed objects
2. **Accept drift**: Use `lifecycle { ignore_changes = [...] }` for fields that are intentionally managed outside Terraform
3. **Import drift**: Use `terraform import` to bring externally-created objects under management

---

## Combining Terraform with Other Tools

A common pattern is using Terraform for NetBox resource creation alongside other providers:

```hcl
# Create IP allocation in NetBox
resource "netbox_available_ip_address" "vm_ip" {
  prefix_id = data.netbox_prefix.servers.id
  status    = "active"
}

# Use the allocated IP in a cloud provider
resource "aws_instance" "server" {
  ami           = "ami-xxxxx"
  instance_type = "t3.micro"
  private_ip    = netbox_available_ip_address.vm_ip.ip_address
}
```

This keeps NetBox and actual infrastructure in sync through a single Terraform state.

---

## Common Gotchas

1. **Version coupling**: The provider's API calls must match the NetBox version exactly. Always test provider upgrades against your NetBox version in a staging environment.

2. **Interface resource names**: in v5.x `netbox_interface` manages **VM** interfaces and `netbox_device_interface` manages device interfaces. v6.0 renames `netbox_interface` to `netbox_virtual_machine_interface`; existing state must be re-imported.

3. **`available_*` failures**: If no IPs/prefixes are available in the parent, the apply fails. Ensure sufficient capacity before running.

4. **Incomplete coverage**: The v5.x provider has strongest support for IPAM and virtualization. For models without Terraform resources, use Ansible or direct API calls. (v6.0 generates resources from the API spec and aims for full coverage.)

5. **Services on NetBox 4.7**: `netbox_service` still writes `protocol` + `ports`; 4.7 accepts them (translated to `port_mappings`) and removes them in 5.0. Services that expose one port on several protocols cannot be expressed by the v5.x resource — manage them via API until the provider adds `port_mappings`.
