# API Rendering Patterns

Detailed examples for rendering configs via the NetBox API using `requests` and `pynetbox`.

## Device Config Rendering

### With requests

```python
import requests

NETBOX = "https://netbox.example.com"
HEADERS = {
    "Authorization": "Bearer nbt_abc123.xxxxxxxxxxxxxxxx",
    "Content-Type": "application/json",
}

# Render config for a specific device
response = requests.post(
    f"{NETBOX}/api/dcim/devices/42/render-config/",
    headers=HEADERS,
    json={},
)
response.raise_for_status()

result = response.json()
config_text = result["content"]
template_info = result["configtemplate"]  # template metadata
print(config_text)
```

### With pynetbox

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token="nbt_abc123.xxxxxxxxxxxxxxxx")

device = nb.dcim.devices.get(name="switch01")
config = device.render_config()  # POST to render-config endpoint
print(config.content)
```

### Raw Text Output

Request `text/plain` to get config without JSON wrapping:

```python
response = requests.post(
    f"{NETBOX}/api/dcim/devices/42/render-config/",
    headers={**HEADERS, "Accept": "text/plain"},
    json={},
)
# response.text is the raw config, no JSON parsing needed
print(response.text)
```

## Passing Extra Context Data

POST body data merges after config context data (overriding on collision):

```python
# Override or supplement config context values at render time
response = requests.post(
    f"{NETBOX}/api/dcim/devices/42/render-config/",
    headers=HEADERS,
    json={
        "ntp_servers": ["10.99.99.1"],        # overrides config context
        "deploy_timestamp": "2026-04-18T12:00", # new variable for template
    },
)
```

This is useful for:
- CI/CD pipelines injecting build-time variables
- Dry-run rendering with test data
- Overriding values without modifying config contexts

## VM Config Rendering

Same pattern, different endpoint:

```python
response = requests.post(
    f"{NETBOX}/api/virtualization/virtual-machines/15/render-config/",
    headers=HEADERS,
    json={},
)
response.raise_for_status()
config_text = response.json()["content"]
```

Template variable is `virtualmachine` (not `device`) inside the template.

## General-Purpose Template Rendering

Render any config template without a device/VM context:

```python
# Useful for generating non-device configs (ACLs, monitoring, etc.)
response = requests.post(
    f"{NETBOX}/api/extras/config-templates/5/render/",
    headers=HEADERS,
    json={
        "region": "us-east",
        "vlans": [10, 20, 30],
        "description": "Auto-generated VLAN config",
    },
)
response.raise_for_status()
print(response.json()["content"])
```

In this mode, the template has access to all NetBox model classes for ORM queries but no `device` or `virtualmachine` variable.

## Bulk Rendering Pattern

Render configs for multiple devices in a loop:

```python
import pynetbox

nb = pynetbox.api("https://netbox.example.com", token="nbt_abc123.xxxxxxxxxxxxxxxx")

# Get all active devices with a specific role
devices = nb.dcim.devices.filter(role="access-switch", status="active")

configs = {}
for device in devices:
    try:
        result = device.render_config()
        configs[device.name] = result.content
    except pynetbox.RequestError as e:
        if e.req.status_code == 400:
            # No config template assigned (or template error)
            print(f"WARNING: {device.name}: {e.error}")
        else:
            raise

# Write configs to files
for name, config in configs.items():
    with open(f"configs/{name}.cfg", "w") as f:
        f.write(config)
```

## Error Handling

### No Template Assigned

If no config template is found (device, role, and platform all lack one):

```
HTTP 400: "No config template has been assigned for this device."
```

### Template Rendering Error

Jinja2 errors (undefined variables, syntax errors) return HTTP 400 with the error message:

```python
response = requests.post(
    f"{NETBOX}/api/dcim/devices/42/render-config/",
    headers=HEADERS,
    json={},
)
if response.status_code == 400:
    print(f"Render error: {response.json()}")
    # Common: UndefinedError when template uses a variable not in context
```

### Permissions

| Endpoint | Required Permission |
|----------|-------------------|
| Device render | `dcim.view_device` + `extras.view_configtemplate` |
| VM render | `virtualization.view_virtualmachine` + `extras.view_configtemplate` |
| Template render | `extras.view_configtemplate` |

## Config Template CRUD

### Create and Assign a Template

```python
# Create template
template = requests.post(
    f"{NETBOX}/api/extras/config-templates/",
    headers=HEADERS,
    json={
        "name": "IOS-XE Base",
        "template_code": "hostname {{ device.name }}\n!\n{% for s in ntp_servers | default([]) %}\nntp server {{ s }}\n{% endfor %}",
        "environment_params": {"undefined": "jinja2.StrictUndefined", "trim_blocks": True},
    },
)
template.raise_for_status()

# Assign to a platform (all devices on this platform inherit the template)
requests.patch(f"{NETBOX}/api/dcim/platforms/3/", headers=HEADERS,
               json={"config_template": template.json()["id"]}).raise_for_status()
```

For template inheritance (`extends`/`include`), templates must be backed by a DataSource — link via `data_source` and `data_file` fields instead of inline `template_code`.
