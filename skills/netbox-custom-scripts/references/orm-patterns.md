# ORM Patterns for Custom Scripts

Scripts have full Django ORM access. All NetBox models are directly importable.

## Creating Objects

```python
from dcim.models import Device

device = Device(
    name='new-switch-01',
    site=site,
    role=role,
    device_type=device_type,
    status='planned',
)
device.full_clean()   # ALWAYS validate
device.save()
self.log_success(f"Created {device}", device)
```

## Updating Objects

```python
device = Device.objects.get(name='existing-switch')
device.snapshot()                    # REQUIRED for change log diff
device.status = 'active'
device._changelog_message = 'Activated via provisioning script'  # Optional
device.full_clean()
device.save()
self.log_success(f"Updated {device}", device)
```

## Deleting Objects

```python
device = Device.objects.get(name='decommissioned')
device.snapshot()    # For change log
device.delete()
self.log_warning(f"Deleted {device}")
```

## get_or_create Pattern

```python
device, created = Device.objects.get_or_create(
    name=hostname,
    defaults={
        'site': site,
        'role': role,
        'device_type': dtype,
        'status': 'planned',
    }
)
if created:
    self.log_success(f"Created {device}", device)
else:
    self.log_info(f"Already exists: {device}", device)
```

> **Note**: `get_or_create` does not call `full_clean()`. Validate separately if using non-trivial defaults.

## Bulk Operations

### Efficient Querying

```python
# Prefetch related objects to avoid N+1 queries
devices = Device.objects.filter(
    site=data['site'],
    status='active',
).select_related(
    'device_type__manufacturer',
    'role',
).prefetch_related(
    'interfaces',
)

for device in devices:
    # device.device_type.manufacturer — no extra query
    # device.interfaces.all() — no extra query
    pass
```

### Bulk Update Pattern

```python
devices = Device.objects.filter(site=data['site'])
count = 0
for device in devices:
    device.snapshot()
    device.status = data['new_status']
    device.full_clean()
    device.save()
    self.log_success(f"Updated {device}", device)
    count += 1
self.log_info(f"Updated {count} devices total")
```

> **No Django `bulk_update()`**: While technically possible, `bulk_update()` bypasses `full_clean()`, `snapshot()`, and change logging. Always use individual save loops in scripts for data integrity and audit trail.

## Counting and Aggregation

```python
from django.db.models import Count, Q

# Count devices per site
sites = Site.objects.annotate(
    active_devices=Count('devices', filter=Q(devices__status='active')),
    total_devices=Count('devices'),
)
for site in sites:
    self.log_info(f"{site.name}: {site.active_devices}/{site.total_devices} active")
```

## Filtering Tips

```python
# Common lookups
Device.objects.filter(name__startswith='nyc-')
Device.objects.filter(status__in=['active', 'staged'])
Device.objects.filter(primary_ip4__isnull=True)
Device.objects.filter(site__region__name='US East')
Device.objects.exclude(role__name='patch-panel')

# Chaining
Device.objects.filter(site=site).exclude(status='decommissioning').order_by('name')
```

## Custom Field Data

Inside a script, custom field values are read and written on `custom_field_data` as **raw stored values** on every NetBox version:

```python
device.custom_field_data['environment']          # 'prod'  (raw value, not {'value': ..., 'label': ...})
device.snapshot()
device.custom_field_data['environment'] = 'staging'
device.full_clean()   # validates against the CustomField definition
device.save()
```

> **NetBox 4.7+**: the REST and GraphQL APIs return selection / multi-selection values as `{"value": "prod", "label": "Production"}` objects. That is serialization only — the ORM value is still `'prod'`. If a script consumes API payloads (e.g. from a webhook or an HTTP call), unwrap `['value']` before comparing with ORM data.

Enumerating the fields defined for a model:

```python
from extras.models import CustomField

# 4.7+: returns a list of *active* CustomField objects (no queryset methods)
# 4.5/4.6: returns a queryset
fields = CustomField.objects.get_for_model(Device)
required = [cf for cf in fields if cf.required]            # works on every version
# NOT: CustomField.objects.get_for_model(Device).filter(required=True)  -> AttributeError on 4.7
```

`device.custom_fields` behaves the same way (list on 4.7). On 4.7 a field whose default is still being provisioned, or which is being deleted, by a background job (`status` = `provisioning` / `deleting`) is omitted from `get_for_model()` and from `custom_fields` until the job finishes; pass `statuses=` to `get_for_model()` to include them.

## Hierarchical Models (Region, SiteGroup, Location, DeviceRole, Platform, TenantGroup, …)

```python
region = Region.objects.get(slug='americas')
for child in region.get_children():        ...
for r in region.get_descendants(include_self=True):   ...
for r in region.get_ancestors():           ...
depth = region.level                       # Python property on every version
```

> **NetBox 4.7+ (ltree)**: `level`, `lft`, `rght`, `tree_id` are no longer database columns — `Region.objects.filter(level=0)` and `.order_by('level')` raise `FieldError`. Use `parent__isnull=True` for roots and `get_ancestors()`/`get_descendants()`/`get_children()` for traversal. `get_root()`, `get_family()`, `is_leaf_node()`, `move_to()`, and `insert_at()` do not exist (reassign `parent` and `save()` to move a node; `not obj.get_children().exists()` for leaf checks). On 4.5/4.6 (django-mptt) the full MPTT API is available, but stick to the subset above so scripts survive the upgrade. After a bulk `COPY`/`UPDATE` that bypassed triggers, run `manage.py rebuild_ltree_paths --check`.

## Search Index Lag (NetBox 4.7+)

Global search index updates are deferred to a background job on 4.7. An object your script just created is immediately visible via the ORM and REST filters, but may not appear in the UI global search (`/search/?q=…`) for a short period. Do not write scripts that create an object and then locate it through the search index in the same run.

## User Context

```python
def run(self, data, commit):
    user = self.request.user  # Current user
    self.log_info(f"Script run by {user.username}")
    # Note: self.request is NetBoxFakeRequest for CLI/scheduled runs
```
