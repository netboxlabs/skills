# Config Context Merging

Deep dive on how NetBox resolves and merges config context data for devices and VMs.

## Assignment Matching

A config context defines assignment criteria across multiple categories. The matching logic:

- **Across categories:** AND — the object must satisfy all non-empty categories
- **Within a category:** OR — the object must match any one of the assigned values

### Example

A config context assigned to:
- Sites: `nyc-dc1`, `lax-dc1`
- Roles: `access-switch`

Matches devices that are in (`nyc-dc1` OR `lax-dc1`) AND have role `access-switch`.

A config context with no site assignment but role `access-switch` matches **all** access switches regardless of site.

### Hierarchy Awareness

For hierarchical models, matching includes all ancestors:

| Model | Hierarchy |
|-------|-----------|
| Region | Region → parent Region |
| Site Group | Site Group → parent Site Group |
| Location | Location → parent Location |

A config context assigned to region "Americas" matches devices in "US-East" (child of "Americas"). This is resolved via `get_ancestors(include_self=True)`.

## Merge Algorithm

### Step 1: Collect Matching Contexts

All **active** (`is_active=True`) config contexts matching the object are collected.

### Step 2: Order

Contexts are ordered by:
1. **Weight** — ascending (lower weight merges first)
2. **Name** — alphabetical (tie-breaker)

Default weight is 1000.

### Step 3: Sequential Deep Merge

Each context's `data` (JSON) is merged into the accumulator using `deepmerge()`:

```
Result = {} 
  ← merge(weight=100 context data)
  ← merge(weight=500 context data)
  ← merge(weight=1000 context data)
  ← merge(local context data)        # always last, always wins
```

### deepmerge() Behavior

```python
def deepmerge(original, new):
    for key, value in new.items():
        if key in original and isinstance(original[key], dict) and isinstance(value, dict):
            deepmerge(original[key], value)  # recursive merge for dicts
        else:
            original[key] = value            # replace everything else
    return original
```

**Key implications:**

| Original Value | New Value | Result | Why |
|---------------|-----------|--------|-----|
| `{"a": 1}` | `{"b": 2}` | `{"a": 1, "b": 2}` | Dict merge |
| `{"a": {"x": 1}}` | `{"a": {"y": 2}}` | `{"a": {"x": 1, "y": 2}}` | Recursive dict merge |
| `[1, 2, 3]` | `[4, 5]` | `[4, 5]` | **List replaced** |
| `"hello"` | `"world"` | `"world"` | Scalar replaced |
| `{"a": [1]}` | `{"a": [2]}` | `{"a": [2]}` | **List in dict replaced** |

## Practical Patterns

### Pattern: Global Defaults with Site Overrides

```
Config Context: "global-ntp" (weight: 100)
  Assignment: (none — matches everything)
  Data: {"ntp_servers": ["10.0.0.1", "10.0.0.2"]}

Config Context: "nyc-ntp" (weight: 200)
  Assignment: Site = nyc-dc1
  Data: {"ntp_servers": ["10.1.0.1"]}
```

NYC devices get `["10.1.0.1"]` (replaced, not appended). All others get `["10.0.0.1", "10.0.0.2"]`.

### Pattern: Additive Data with Dict Keys

To achieve additive merging, use dicts instead of lists:

```json
// Weight 100: global
{"dns_servers": {"primary": "8.8.8.8", "secondary": "8.8.4.4"}}

// Weight 200: site override — adds tertiary without losing primary/secondary
{"dns_servers": {"tertiary": "10.1.0.53"}}

// Result (dict merge is recursive):
{"dns_servers": {"primary": "8.8.8.8", "secondary": "8.8.4.4", "tertiary": "10.1.0.53"}}
```

### Pattern: Role-Specific Configuration

```
Config Context: "base-snmp" (weight: 100)
  Assignment: (none)
  Data: {"snmp": {"community": "public", "version": "2c"}}

Config Context: "core-snmp" (weight: 200)
  Assignment: Role = core-router
  Data: {"snmp": {"community": "s3cur3", "contact": "noc@example.com"}}

// Core routers get:
{"snmp": {"community": "s3cur3", "version": "2c", "contact": "noc@example.com"}}
```

### Local Context Always Wins

Data defined directly on a device/VM (`local_context_data` field) merges last:

```
All config contexts merged → {"ntp_servers": ["10.0.0.1"]}
Device local context       → {"ntp_servers": ["192.168.1.1"]}
Final result               → {"ntp_servers": ["192.168.1.1"]}
```

Use local context sparingly — for true per-device exceptions only.

## Performance Considerations

### Bulk Operations

When querying many devices, config context computation per-device is expensive. NetBox can annotate querysets with `jsonb_agg` for bulk computation via PostgreSQL, but the default API list endpoint computes individually.

**Always use `?exclude=config_context`** on device/VM list endpoints when you don't need the merged config context:

```python
# Slow — computes config context for every device
devices = requests.get(f"{NETBOX}/api/dcim/devices/", headers=headers)

# Fast — skips config context computation  
devices = requests.get(f"{NETBOX}/api/dcim/devices/?exclude=config_context", headers=headers)
```

### Config Context Profiles (Schema Validation)

Config context profiles define a JSON Schema that context data must conform to. Use these to enforce consistency:

- Prevent typos in key names
- Enforce required fields
- Validate data types
- Profiles can be synced from DataSources (git-managed schemas)

## Debugging Merge Results

To see the final merged config context for a device:

```python
# Via API — includes the fully merged result
device = requests.get(f"{NETBOX}/api/dcim/devices/42/", headers=headers)
merged = device.json()["config_context"]  # final merged dict

# Compare with individual contexts
contexts = requests.get(
    f"{NETBOX}/api/extras/config-contexts/?device_id=42",
    headers=headers,
)
for ctx in contexts.json()["results"]:
    print(f"{ctx['name']} (weight {ctx['weight']}): {ctx['data']}")
```
