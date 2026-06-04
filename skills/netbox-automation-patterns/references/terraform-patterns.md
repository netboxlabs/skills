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

| Provider Version | NetBox Version |
|-----------------|---------------|
| v5.0.0+ | 4.3–4.4.x |

> The community Terraform provider (e-breuninger/terraform-provider-netbox) trails NetBox releases. For **NetBox 4.5/4.6** do not assume the row above still applies — check the provider's current release notes / compatibility matrix and pin to the version that lists support for your exact NetBox minor. NetBox can introduce breaking API changes in minor releases, so a mismatched provider fails in subtle ways.

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
| Virtualization | `netbox_cluster`, `netbox_virtual_machine`, `netbox_vm_interface` | Yes |
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

2. **`netbox_interface` renamed**: The old `netbox_interface` resource was renamed to `netbox_device_interface`. Existing state requires migration.

3. **`available_*` failures**: If no IPs/prefixes are available in the parent, the apply fails. Ensure sufficient capacity before running.

4. **Incomplete coverage**: The provider has strongest support for IPAM and virtualization. For models without Terraform resources, use Ansible or direct API calls.
