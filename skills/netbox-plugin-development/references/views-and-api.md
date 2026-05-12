# Views and API Patterns

## Generic View Classes

| View Class | Purpose | Required Attributes |
|-----------|---------|-------------------|
| `ObjectListView` | List/filter objects | `queryset`, `table`, `filterset`, `filterset_form` |
| `ObjectView` | Detail view | `queryset` |
| `ObjectEditView` | Create/edit form | `queryset`, `form` |
| `ObjectDeleteView` | Delete confirmation | `queryset` |
| `BulkImportView` | CSV import | `queryset`, `model_form` |
| `BulkEditView` | Edit multiple | `queryset`, `table`, `filterset`, `form` |
| `BulkDeleteView` | Delete multiple | `queryset`, `table`, `filterset` |
| `ObjectChildrenView` | Related objects tab | `queryset`, `child_model`, `table`, `tab` |

## @register_model_view Decorator

Registers views with NetBox's URL system — eliminates manual URL patterns:

```python
from utilities.views import register_model_view

@register_model_view(AccessList, 'list')      # /plugins/myplugin/access-lists/
@register_model_view(AccessList)               # /plugins/myplugin/access-lists/<pk>/
@register_model_view(AccessList, 'edit')       # /plugins/myplugin/access-lists/<pk>/edit/
@register_model_view(AccessList, 'delete')     # /plugins/myplugin/access-lists/<pk>/delete/
```

Then in `urls.py`:
```python
from netbox.views.generic import get_model_urls
from . import models
urlpatterns = get_model_urls('netbox_myplugin', models)
```

## URL Namespace Conventions

| Context | Pattern | Example |
|---------|---------|---------|
| UI views | `plugins:<plugin>:<view>` | `plugins:netbox_myplugin:accesslist_list` |
| API detail | `plugins-api:<plugin>-api:<basename>-detail` | `plugins-api:netbox_myplugin-api:accesslist-detail` |
| API list | `plugins-api:<plugin>-api:<basename>-list` | `plugins-api:netbox_myplugin-api:accesslist-list` |

## ViewTab — Extending Core Model Views

Add tabs to existing models (core or other plugins):

```python
from utilities.views import register_model_view, ViewTab
from netbox.views.generic import ObjectChildrenView
from dcim.models import Device
from .models import AccessList
from .tables import AccessListTable

@register_model_view(Device, 'access_lists', path='access-lists')
class DeviceAccessListsView(ObjectChildrenView):
    queryset = Device.objects.all()
    child_model = AccessList
    table = AccessListTable
    tab = ViewTab(
        label='Access Lists',
        badge=lambda obj: AccessList.objects.filter(device=obj).count(),
        permission='netbox_myplugin.view_accesslist',
    )

    def get_children(self, request, parent):
        return AccessList.objects.filter(device=parent)
```

## REST API — Serializer Patterns

### Nested/Brief Representation

```python
class AccessListSerializer(NetBoxModelSerializer):
    class Meta:
        model = AccessList
        fields = ('id', 'url', 'display', 'name', 'device', 'type',
                  'tags', 'custom_fields', 'created', 'last_updated')
        brief_fields = ('id', 'url', 'display', 'name')
```

`brief_fields` controls what's shown when this object appears nested inside another
serializer. Always include `id`, `url`, `display` plus the most identifying field.

### Related Object Fields

```python
from netbox.api.fields import ChoiceField, SerializedPKRelatedField

class AccessListSerializer(NetBoxModelSerializer):
    type = ChoiceField(choices=AccessListTypeChoices)
    # For nested representation of related objects, use nested=True:
    device = DeviceSerializer(nested=True, read_only=True)
```

### HyperlinkedIdentityField

If you need to explicitly set the URL field:
```python
from rest_framework.relations import HyperlinkedIdentityField

url = HyperlinkedIdentityField(
    view_name='plugins-api:netbox_myplugin-api:accesslist-detail'
)
```

## REST API — ViewSet & Router

```python
# api/views.py
from netbox.api.viewsets import NetBoxModelViewSet
from ..filtersets import AccessListFilterSet
from .serializers import AccessListSerializer
from ..models import AccessList

class AccessListViewSet(NetBoxModelViewSet):
    queryset = AccessList.objects.all()
    serializer_class = AccessListSerializer
    filterset_class = AccessListFilterSet

# api/urls.py
from netbox.api.routers import NetBoxRouter
from . import views

router = NetBoxRouter()
router.register('access-lists', views.AccessListViewSet)
urlpatterns = router.urls
```

API endpoints appear at `/api/plugins/myplugin/access-lists/`.

## GraphQL API (Strawberry)

```python
# graphql/types.py
import strawberry_django
from netbox_myplugin.models import AccessList

@strawberry_django.type(AccessList)
class AccessListType:
    id: int
    name: str
    device: 'DeviceType'  # forward reference

# graphql/schema.py
import strawberry
import strawberry_django
from .types import AccessListType

@strawberry.type(name="Query")
class Query:
    access_list: AccessListType = strawberry_django.field()
    access_list_list: list[AccessListType] = strawberry_django.field()
```

Schema is auto-discovered at `graphql.schema`. No manual registration needed.

> **4.5 breaking change:** GraphQL filter syntax now requires lookup modifiers for
> ID and enum fields (e.g., `device_id: {exact: 1}` not `device_id: 1`).
