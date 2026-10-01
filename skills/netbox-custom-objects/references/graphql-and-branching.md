# GraphQL and Branching

Two v0.6.0 capabilities that change how integrations read custom objects and how the plugin behaves alongside netbox-branching. Both are version-gated — confirm the plugin version from `/api/status/` first.

## GraphQL (v0.6+)

Custom objects appear in NetBox's standard `/graphql/` endpoint. Auth is the same as REST (`Authorization: Bearer nbt_…`); object-level permissions apply to root queries and to objects reached through relationship fields.

### Root field naming

For each Custom Object Type two root fields exist:

| Field | Purpose |
|-------|---------|
| `custom_objects_<slug>(id: N)` | One object by id |
| `custom_objects_<slug>_list` | Paginated list |

`<slug>` is the COT slug lowercased with any non-identifier character replaced by `_` (slug `dhcp-scopes` → `custom_objects_dhcp_scopes`). The prefix prevents collisions with core root fields (a type slugged `site` or `group` would otherwise be shadowed). If two slugs sanitize to the same name, the later one is disambiguated with the type id — check introspection.

The schema is rebuilt from the database per request, so a type or field added at runtime is queryable immediately without a restart.

### Queries

```graphql
query {
  custom_objects_dhcp_scopes_list {
    id
    display              # display_expression or primary field value
    created
    last_updated
    tags { name }
    scope_name           # one field per COTF, named by field name
    lease_time
    ip_range {           # object field -> native IPRangeType, traversable
      id
      start_address
      vrf { name }
    }
    backup_devices {     # multiobject -> [DeviceType]
      id
      name
    }
    attached_to {        # polymorphic -> union; select per member type
      ... on DeviceType { id name }
      ... on VirtualMachineType { id name }
    }
  }
  custom_objects_dhcp_scopes(id: 42) { id display }
}
```

Every type exposes `id`, `display`, `created`, `last_updated`, and `tags`. `name` exists only if the type defines a field literally named `name`. All custom fields are nullable.

### Type mapping

| Custom field type | GraphQL |
|-------------------|---------|
| text, longtext, url, select | `String` |
| integer | `Int` |
| decimal | `Decimal` |
| boolean | `Boolean` |
| date, datetime | `Date`, `DateTime` |
| json | `JSON` |
| multiselect | `[String]` |
| object, multiobject | target's native type (`SiteType`, `DeviceType`, another COT's type, …) |
| polymorphic object/multiobject | union of member types |
| target without a GraphQL type | `CustomObjectRelatedObjectType { id object_type display url }` |
| coordinates | not exposed (use REST `<name>_latitude` / `<name>_longitude`) |
| url title | not exposed (only the URL string) |

### Limits (as of v0.7.0)

- **Read-only.** No mutations are generated; write via REST.
- **No `filters:` argument** on list fields. Filter server-side with REST, or fetch the list and filter client-side. Lists are paginated with strawberry-django's standard pagination argument.
- The GraphiQL explorer caches the schema when the page loads; after changing types or fields, click **Re-fetch GraphQL schema** (or reload). Programmatic clients always see the current schema.

## Branching

### Version history — read this first

| Plugin | Behavior |
|--------|----------|
| **v0.6.0+** | COTs, fields, and instances are all branch-aware. Field adds/renames/deletes, type metadata changes, type deletion, and instance create/update/delete inside a branch are diffed, merged, and revertible. Portable schema apply works inside a branch. |
| v0.5.x | Instance writes rejected on a branch; COT/COTF edits bypass the branch and land on main; schema apply blocked. |
| v0.4.x | "Limited compatibility" — same as v0.5 without the write guard warnings. |

### Version gate (v0.6+)

When `netbox_branching` is in `PLUGINS`, startup enforces:

- **NetBox ≥ 4.6.2**
- **netbox-branching ≥ 1.0.4**

Failures surface as Django system checks `netbox_custom_objects.E001` (NetBox too old) and `E002` (branching too old); `W001` means the branching version could not be determined (editable install). Without branching installed, the plugin's normal range (NetBox 4.5.2–4.7.x) applies.

```bash
pip install "netboxlabs-netbox-custom-objects[branching]"   # pulls netbox-branching>=1.0.4
```

Pair with the netbox-branching line for your NetBox: **1.2.x on NetBox 4.7**, **1.1.x on 4.4–4.6** (see [netbox-branching](../../netbox-branching/SKILL.md)).

### `exempt_models` — do not carry forward blindly

The installation and branching doc pages still include the pre-0.6 snippet:

```python
PLUGINS_CONFIG = {
    'netbox_branching': {
        'exempt_models': [
            'netbox_custom_objects.customobjecttype',
            'netbox_custom_objects.customobjecttypefield',
        ],
    },
}
```

and the pre-0.6 statement that instance writes on branches are disallowed. That reflected v0.4/v0.5. In v0.6+:

- The branching integration is exercised with **no `exempt_models`** — type and field changes are meant to be tracked in the branch alongside instance changes.
- Exempting the schema models makes type/field edits go straight to main while instance rows stay in the branch, reintroducing the split the 0.6 work removed.

Recommendation: on v0.6+, omit the snippet unless the release notes for your installed version say otherwise, test a branch create → add field → add instance → merge → revert cycle in staging, and treat the doc page's "disallowed" language as stale. On v0.4/v0.5 keep the snippet.

### What to expect on a branch (v0.6+)

- Creating a COT or field in a branch creates it in the branch's schema only; merging applies the DDL to main; reverting drops it again.
- Renaming a field in a branch while instances change on main (or vice versa) syncs and merges; the changelog is rewritten to the new column name.
- Deleting a branch without merging leaks nothing to main.
- The Custom Objects related tab, GraphQL, and REST all read the active branch's data.
- Upgrading the plugin heals each branch's schema (e.g. adds `<url>_title` columns) on `manage.py migrate` or `manage.py upgrade_custom_objects`.

### Config context aggregation rules (v0.6+)

Relevant when modeling types that should inherit ConfigContexts. With `config_context_enabled: true` on the COT:

| Field name | Must point at | Feeds |
|------------|---------------|-------|
| `site` | `dcim.site` | site, region, site group |
| `tenant` | `tenancy.tenant` | tenant, tenant group |
| `role` | `dcim.devicerole` | role |
| `platform` | `dcim.platform` | platform |
| `location` | `dcim.location` | location |
| `device_type` | `dcim.devicetype` | device type |
| `cluster` | `virtualization.cluster` | cluster, cluster type, cluster group |

- The field must be a non-polymorphic `object` field with **both** the name and target above; anything else is ignored.
- Tags on the custom object match tag-scoped contexts automatically.
- `local_context_data` is merged last and wins.
- A type with **none** of these fields receives only its local data — global (unassigned) ConfigContexts are *not* applied, unlike Devices/VMs. Add `site` (or another convention field) to opt in.
- Note `location` here is a **field name** convention pointing at `dcim.location`, unrelated to the `coordinates` field type.
