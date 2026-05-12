# Model Patterns

## Base Class Hierarchy

```
django.db.Model
└── NetBoxModel              # Tags, custom fields, change logging, bookmarks,
    │                        # journaling, export templates, event rules, notifications
    ├── PrimaryModel         # + description, comments fields (4.5+ plugins API)
    │   └── NestedGroupModel # + MPTT tree structure (parent, level, etc.)
    └── OrganizationalModel  # + name (unique), slug (unique), description, comments
```

**Choose your base class:**
- `NetBoxModel` — generic plugin object, you define all fields
- `PrimaryModel` — object with description + comments (e.g., a circuit, a service)
- `OrganizationalModel` — categorization object with unique name/slug (e.g., a role, a type)
- `NestedGroupModel` — hierarchical grouping (e.g., regions, location types)

## NetBoxFeatureSet Mixins

`NetBoxModel` includes all of these automatically:

| Mixin | Feature |
|-------|---------|
| `BookmarksMixin` | User bookmarks |
| `ChangeLoggingMixin` | Change log entries |
| `CloningMixin` | Object cloning |
| `CustomFieldsMixin` | Custom fields |
| `CustomLinksMixin` | Custom links |
| `CustomValidationMixin` | Custom validation rules |
| `ExportTemplatesMixin` | Custom export templates |
| `JournalingMixin` | Journal entries |
| `NotificationsMixin` | User notifications |
| `TagsMixin` | Tag assignment |
| `EventRulesMixin` | Event rules (webhooks, scripts) |

## Choices Pattern

```python
from utilities.choices import ChoiceSet

class AccessListTypeChoices(ChoiceSet):
    key = 'AccessList.type'

    CHOICES = [
        ('standard', 'Standard', 'blue'),   # (value, label, color)
        ('extended', 'Extended', 'orange'),
    ]
```

Use in model field:
```python
type = models.CharField(max_length=50, choices=AccessListTypeChoices)
```

## RestrictedQuerySet

All plugin models using `NetBoxModel` get object-level permissions via
`RestrictedQuerySet`. The default manager handles this automatically. If you
override the manager, inherit from `RestrictedQuerySet`:

```python
from netbox.models import RestrictedQuerySet

class AccessListQuerySet(RestrictedQuerySet):
    def active(self):
        return self.filter(status='active')

class AccessList(NetBoxModel):
    objects = AccessListQuerySet.as_manager()
```

## ForeignKey Best Practices

```python
# Reference core models as strings
device = models.ForeignKey(
    to='dcim.Device',
    on_delete=models.CASCADE,
    related_name='%(app_label)s_access_lists',  # avoids collisions
)

# Nullable FK (optional relationship)
tenant = models.ForeignKey(
    to='tenancy.Tenant',
    on_delete=models.SET_NULL,
    blank=True, null=True,
    related_name='%(app_label)s_access_lists',
)

# GenericForeignKey for polymorphic relations
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType

assigned_object_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
assigned_object_id = models.PositiveBigIntegerField()
assigned_object = GenericForeignKey('assigned_object_type', 'assigned_object_id')
```

## Migration Tips

- Create migrations: `python manage.py makemigrations netbox_myplugin`
- Plugin migrations live in `netbox_myplugin/migrations/`
- When referencing core models in migrations, use `('dcim', 'Device')` tuple
- Test migrations both forward and backward: `python manage.py migrate netbox_myplugin zero`
- Never import model classes directly in migration files — use `apps.get_model()`

## get_absolute_url() Pattern

URL always follows: `plugins:<plugin_name>:<model_name_lower>`

```python
def get_absolute_url(self):
    return reverse('plugins:netbox_myplugin:accesslist', args=[self.pk])
```

Common mistake: forgetting the `plugins:` prefix or using hyphens instead of
underscores in the plugin name portion.
