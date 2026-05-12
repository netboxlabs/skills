---
name: build-plugin
description: Scaffold a new NetBox plugin with standard project structure.
---

# Build a NetBox Plugin

Guide the user through scaffolding a new NetBox plugin. Reference the [netbox-plugin-development](../skills/netbox-plugin-development/SKILL.md) skill for detailed implementation patterns.

## Step 1: Gather Requirements

Ask the user:
1. **Plugin name** (e.g., `netbox-my-plugin`)
2. **What does it do?** (new data models, UI extensions, background jobs, etc.)
3. **Target NetBox version** (4.x recommended)

## Step 2: Scaffold Directory Structure

```
netbox-my-plugin/
├── netbox_my_plugin/
│   ├── __init__.py              # PluginConfig class
│   ├── models.py                # Django models
│   ├── views.py                 # UI views
│   ├── api/
│   │   ├── __init__.py
│   │   ├── serializers.py       # DRF serializers
│   │   ├── views.py             # API viewsets
│   │   └── urls.py              # API URL routing
│   ├── forms.py                 # Django forms
│   ├── filtersets.py            # Filtering logic
│   ├── tables.py                # django-tables2 table classes
│   ├── urls.py                  # UI URL routing
│   ├── navigation.py            # Menu items
│   ├── templates/
│   │   └── netbox_my_plugin/    # Template files
│   └── migrations/
│       └── __init__.py
├── setup.py                     # or pyproject.toml
├── MANIFEST.in
├── README.md
└── requirements.txt
```

## Step 3: PluginConfig

```python
from netbox.plugins import PluginConfig

class MyPluginConfig(PluginConfig):
    name = 'netbox_my_plugin'
    verbose_name = 'My Plugin'
    description = 'Description of what this plugin does'
    version = '0.1.0'
    base_url = 'my-plugin'
    min_version = '4.0.0'
```

## Step 4: Decision Table

| Need | Approach | Key Files |
|------|----------|-----------|
| New data model | Django model + migration | `models.py`, `migrations/` |
| Custom UI pages | Views + templates | `views.py`, `templates/` |
| REST API endpoints | DRF viewsets | `api/views.py`, `api/serializers.py` |
| Navigation menu items | PluginMenuItem | `navigation.py` |
| Background processing | NetBox Job framework | `jobs.py` |
| Template extensions | TemplateExtension class | `template_content.py` |
| Custom validators | Validator classes | `validators.py` |
| Event-driven actions | Event rules | `event_rules.py` |

## Step 5: Install and Test

```bash
# Development install
pip install -e .

# Add to NetBox configuration.py
PLUGINS = ['netbox_my_plugin']

# Run migrations
cd /opt/netbox/netbox
python manage.py migrate

# Restart services
sudo systemctl restart netbox netbox-rq
```

## Example Invocations

- "Build a plugin to track maintenance windows for devices"
- "Scaffold a plugin that adds a contracts model linked to tenants"
- "Create a plugin with a background job that syncs data from ServiceNow"

For detailed implementation patterns (model mixins, API serializers, testing, packaging), load the `netbox-plugin-development` skill.
