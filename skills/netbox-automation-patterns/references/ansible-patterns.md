# Ansible Integration Patterns

Collection: `netbox.netbox` (GPLv3) — install via `ansible-galaxy collection install netbox.netbox`.

**Requirements:** Python 3.11+, pynetbox, pytz, Ansible 2.18+. Supports the two most recent NetBox releases.

---

## Module Patterns (State Management)

All modules follow a consistent interface:

```yaml
- name: Ensure site exists
  netbox.netbox.netbox_site:
    netbox_url: "{{ netbox_url }}"
    netbox_token: "{{ netbox_token }}"
    data:
      name: DC1
      slug: dc1
      status: active
      region: US East
      tags:
        - production
    state: present  # default; use "absent" to delete
```

### Key Properties

- **Idempotent**: Running the same task twice produces no change on the second run
- **Object identification**: Objects are matched by unique fields (name, slug, etc.)
- **`state: present`** creates if missing, updates if different
- **`state: absent`** deletes if exists, no-op if already gone
- **Nested references**: Related objects can be referenced by name (e.g., `region: US East`)

### Available Module Categories

| Category | Key Modules |
|----------|------------|
| DCIM | `netbox_device`, `netbox_site`, `netbox_rack`, `netbox_device_interface`, `netbox_cable`, `netbox_platform`, `netbox_manufacturer` |
| IPAM | `netbox_ip_address`, `netbox_prefix`, `netbox_vlan`, `netbox_vrf`, `netbox_aggregate` |
| Virtualization | `netbox_cluster`, `netbox_virtual_machine`, `netbox_vm_interface` |
| Circuits | `netbox_circuit`, `netbox_provider`, `netbox_circuit_termination` |
| Tenancy | `netbox_tenant`, `netbox_contact`, `netbox_contact_assignment` |
| Extras | `netbox_tag`, `netbox_custom_field`, `netbox_config_context`, `netbox_webhook`, `netbox_event_rule` |

### Bulk Operations Pattern

Use loops for bulk state management:

```yaml
- name: Ensure all sites exist
  netbox.netbox.netbox_site:
    netbox_url: "{{ netbox_url }}"
    netbox_token: "{{ netbox_token }}"
    data: "{{ item }}"
    state: present
  loop: "{{ sites }}"
  loop_control:
    label: "{{ item.name }}"
```

Define sites in a variable file:
```yaml
sites:
  - name: DC1
    slug: dc1
    status: active
    region: US East
  - name: DC2
    slug: dc2
    status: active
    region: US West
```

---

## Dynamic Inventory (`nb_inventory`)

Create an inventory file (e.g., `netbox_inventory.yml`):

```yaml
plugin: netbox.netbox.nb_inventory
api_endpoint: https://netbox.example.com
token: nbt_abc123.xxxxxxxxxxxxxxxx
validate_certs: true

# Performance: disable config context fetching unless needed
config_context: false

# Group hosts by these attributes
group_by:
  - device_roles
  - sites
  - tags
  - platforms

# Server-side filtering (reduces API calls)
query_filters:
  - role: network-edge-router
  - status: active

# Additional device-specific filters
device_query_filters:
  - has_primary_ip: 'true'
```

### Performance Tips

1. **Set `config_context: false`** — Config context fetching adds a separate API call per device. Only enable if your playbooks consume config context data.

2. **Use `query_filters`** — Filter server-side to avoid fetching thousands of devices you don't need.

3. **Use `device_query_filters`** — Filter for devices with primary IPs (`has_primary_ip: 'true'`) to avoid inventory entries you can't connect to.

4. **Limit `group_by`** — Each grouping dimension adds processing overhead. Only group by what your playbooks actually use.

### Testing Inventory

```bash
# List all hosts
ansible-inventory -i netbox_inventory.yml --list

# Show details for a specific host
ansible-inventory -i netbox_inventory.yml --host router01
```

---

## Lookup Plugin (`nb_lookup`)

Query NetBox data within playbooks:

```yaml
- name: Get all active devices
  set_fact:
    devices: "{{ query('netbox.netbox.nb_lookup', 'devices',
                        api_endpoint='https://netbox.example.com',
                        token='nbt_abc123.xxxxxxxxxxxxxxxx',
                        api_filter='status=active') }}"

- name: Show device names
  debug:
    msg: "{{ item.value.display }}"
  loop: "{{ devices }}"
```

Use `nb_lookup` for ad-hoc queries when the inventory plugin doesn't provide the data structure you need.

---

## Common Gotchas

1. **pynetbox version mismatch**: Ensure your pynetbox version is compatible with your NetBox version. Incompatible versions cause cryptic serialization errors.

2. **Token permissions**: Module tasks that create/update objects need a write-capable token. Inventory and lookup only need read access. Use separate tokens with minimal scope.

3. **Large inventories are slow**: The inventory plugin generates many host variables per device. For inventories with thousands of devices, `config_context: false` and tight `query_filters` are essential.

4. **Collection version pinning**: Pin the collection version in `requirements.yml` to avoid breaking changes:
   ```yaml
   collections:
     - name: netbox.netbox
       version: ">=3.20.0,<4.0.0"   # example — verify against the release that supports your NetBox 4.5/4.6
   ```
   The literal above is illustrative. The `netbox.netbox` collection supports the two most recent NetBox releases, so confirm the version (and its NetBox support matrix) for your target 4.5/4.6 minor rather than copying a fixed pin.
