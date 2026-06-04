# Script Variables Reference

All variables inherit common options: `label`, `description`, `default`, `required` (default `True`), `widget`.

## Variable Types

### StringVar

```python
name = StringVar(
    label="Device Name",
    min_length=3,
    max_length=64,
    regex=r'^[a-z]',       # Regex validation
    description="Must start with lowercase letter",
)
```

### TextVar

Multi-line textarea. No additional options beyond base.

```python
notes = TextVar(label="Notes", required=False)
```

### IntegerVar / DecimalVar

```python
count = IntegerVar(min_value=1, max_value=100, default=10)
weight = DecimalVar(min_value=0, max_digits=5, decimal_places=2)
```

### BooleanVar

Always `required=False` internally (unchecked = `False`).

```python
dry_run = BooleanVar(label="Dry Run", default=True, description="Preview changes only")
```

### ChoiceVar / MultiChoiceVar

```python
action = ChoiceVar(
    choices=(
        ('enable', 'Enable'),
        ('disable', 'Disable'),
        ('delete', 'Delete'),
    ),
    label="Action",
)

tags = MultiChoiceVar(
    choices=(('prod', 'Production'), ('dev', 'Development'), ('staging', 'Staging')),
    label="Environments",
)
```

A blank choice is automatically prepended to `ChoiceVar`.

### ObjectVar

API-backed dynamic dropdown for any NetBox model.

```python
from dcim.models import Site, Device

site = ObjectVar(
    model=Site,
    query_params={'status': 'active'},    # Static filter
    null_option='(Any)',                   # Optional "none" choice
    description="Select a site",
)
```

**Dynamic filtering** — reference other form fields with `$`:

```python
region = ObjectVar(model=Region)
site = ObjectVar(model=Site, query_params={'region_id': '$region'})
device = ObjectVar(model=Device, query_params={'site_id': '$site', 'status': 'active'})
```

**Context customization** for dropdown rendering:

```python
site = ObjectVar(
    model=Site,
    context={
        'value': 'id',              # Default
        'label': 'display',         # Default
        'description': 'description',
        'depth': '_depth',          # For hierarchical models
    }
)
```

**Picker options:**

```python
device = ObjectVar(
    model=Device,
    selector=True,      # 4.5+: advanced object-selection widget (filterable picker dialog)
    quick_add=True,     # 4.6.2+: inline "create new" button to add a missing object without leaving the form
)
```

### MultiObjectVar

Same options as `ObjectVar`, allows multiple selection.

```python
devices = MultiObjectVar(model=Device, query_params={'site_id': '$site'})
```

### FileVar

File upload — **only available during script execution**. Process immediately in `run()`.

```python
csv_file = FileVar(label="CSV Import File", description="Upload a CSV file")

def run(self, data, commit):
    content = data['csv_file'].read().decode('utf-8')
    # Process content...
```

### IP Address Variables

```python
host_ip = IPAddressVar(label="Host IP")                    # e.g., 192.168.1.1
interface_ip = IPAddressWithMaskVar(label="Interface IP")  # e.g., 192.168.1.1/24
network = IPNetworkVar(
    label="Network",
    min_prefix_length=16,
    max_prefix_length=28,
)  # e.g., 10.0.0.0/24
```

### DateVar / DateTimeVar

```python
start_date = DateVar(label="Start Date")
scheduled_at = DateTimeVar(label="Scheduled Time", required=False)
```

## Tips

- Set `required=False` for optional fields (except `BooleanVar` which is always optional)
- Use `default` to pre-populate commonly used values
- Use `regex` on `StringVar` for input validation without custom code
- Chain `ObjectVar` fields with `$` references for cascading dropdowns
- `ChoiceVar` values are always strings — cast in `run()` if needed
