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

## User Context

```python
def run(self, data, commit):
    user = self.request.user  # Current user
    self.log_info(f"Script run by {user.username}")
    # Note: self.request is NetBoxFakeRequest for CLI/scheduled runs
```
