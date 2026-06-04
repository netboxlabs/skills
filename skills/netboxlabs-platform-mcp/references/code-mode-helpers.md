# Code-Mode Injected Helpers

These functions are injected into the `netbox_execute` namespace. The **live** `netbox_execute` tool description is authoritative for your deployment — helpers are gated by capability, so a gated helper simply won't appear. Object types use dot notation (`'dcim.device'`); the path form (`'dcim/devices'`) also resolves.

In `netbox_search_schema`, you don't call these — instead you read and filter the pre-loaded `schema` variable (sections: `object_types`, `operations`, `special_endpoints`, `filters`, `graphql`, `tips`, `plugins`, `introspection`, plus per-product sections when entitled).

## Baseline reads (always present)

```
get(type, filters={}, fields=[], limit=50, offset=0, ordering=None) → {count, next, previous, results[]}
get_by_id(type, id, fields=[]) → object dict
search(query, object_types=[], limit=5) → {type: [results]}
trace(type, id) → cable path segments (single endpoint → far end)
cable_paths(type, id) → all cable paths through a pass-through port
connected_device(peer_device, peer_interface) → device on the far side of a cable
rack_elevation(rack_id, face='front') → {total_units, occupied, free, units[]}
options(type) → field schema (required fields, choices)
status() → NetBox version, plugins, workers
available_prefixes(prefix_id, limit=10) → available prefixes
available_ips(prefix_id, limit=10) → available IPs
available_ips_in_range(ip_range_id, limit=10) → available IPs in a range
available_vlans(group_id, limit=10) → available VLANs
available_asns(asn_range_id, limit=10) → available ASNs
changelogs(filters={}, limit=50, offset=0) → audit-trail entries
custom_field_choices(custom_field_id) → [{value, display}]
discover_models(app=None, include_plugin_detail=False) → apps/endpoints (core + plugins)
inspect_model(app, endpoint) → full field schema from OPTIONS
```

## WRITE-gated

```
create(type, data) → created object with ID
update(type, id, data) → updated object (PATCH semantics)
delete(type, id) → True
bulk_create(type, [data]) → list of created objects
bulk_update(type, [{id, ...fields}]) → list of updated objects
bulk_delete(type, [ids]) → True
sync_data_source(data_source_id) → data source with refreshed sync status
```

## IPAM_ALLOCATION-gated

```
allocate_prefix(prefix_id, prefix_length, **kwargs) → new prefix
allocate_ip(prefix_id, **kwargs) → new IP
allocate_ip_in_range(ip_range_id, **kwargs) → new IP from a range
allocate_vlan(group_id, name, **kwargs) → new VLAN
allocate_asn(asn_range_id, **kwargs) → new ASN
```

## Other capability gates

```
render_config(type, id, context=None) → rendered config string      # CONFIG_RENDER
run_script(script_id, data=None) → script detail with job info       # SCRIPTS (async; poll job)
graphql(query, variables=None, analyze=True) → {data, errors, analysis}   # GRAPHQL (auto cost-analysis)
graphql_schema_info(type_name=None) → introspect schema              # GRAPHQL
api(method, endpoint, params=None, data=None) → raw API call         # always present; GET-only if WRITE absent
```

## Plugin helpers (when the plugin is installed)

**Branching** (`PLUGIN_BRANCHING`):
```
branch_list(status=None, limit=50, offset=0) / branch_create(name, description=None)
branch_sync(branch_id, commit=False) / branch_merge(branch_id, commit=False)
branch_revert(branch_id, commit=False) / branch_archive(branch_id)
branch_diffs(branch_id=None, object_type=None, action=None, has_conflicts=None, limit=50, offset=0)
```

**Changes** (`PLUGIN_CHANGES`):
```
change_request_list / change_request_get / change_request_create(name, branch, policy, ...)
change_request_update / submit_review(change_request, status, comments=None)
list_reviews / list_policies / add_comment(change_request, content) / reply_to_comment(comment, content)
```

**Custom Objects** (`PLUGIN_CUSTOM_OBJECTS`):
```
custom_type_list / custom_type_schema(type_id)
custom_object_list(custom_type_slug, filters={}, ...) / custom_object_get(custom_type_slug, object_id)
custom_object_create / custom_object_update / custom_object_delete
```

## Product helpers (entitlement-gated)

Appear in `netbox_execute`'s description and under their own `schema` section when the tenant is entitled and the integration is configured. Frame as "available when your tenant is entitled to Assurance/Discovery."

**Assurance** (`assurance_*`): `get_deviation_types`, `get_deviations`, `get_deviation`, `action_deviation`, `bulk_action`, `timeseries`, `list_entities`, `create_entity`. (Write helpers gated.)

**Discovery** (`discovery_*`): `list_agents`, `get_agent`, `create_agent`, `edit_agent`, `delete_agent`, `list_groups`, `create_group`, `get_group`, `list_policies`, `get_policy`, `create_policy`, `list_datasets`, `create_dataset`, `list_runs`, `list_labels`. (Write helpers gated.)

A Validation plugin contributes its own sandbox helpers and a `validation` schema section when installed.
