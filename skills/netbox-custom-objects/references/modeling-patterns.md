# Modeling Patterns

Common patterns for designing custom object types (plugin v0.7.x).

## Before Creating a Custom Object

Ask these questions first:

1. **Does a built-in NetBox model already cover this?** Check DCIM, IPAM, Circuits, Tenancy, Virtualization first (see [netbox-data-modeling](../../netbox-data-modeling/SKILL.md)).
2. **Would custom fields on an existing model suffice?** If you just need 2-3 extra fields on Device or Site, custom fields are simpler.
3. **Do you need complex business logic?** Custom objects are data-only — validation is limited to field rules plus `CUSTOM_VALIDATORS`. For workflows, consider a plugin.
4. **Will the schema move between environments?** Give the type a `version` and plan to manage it with the portable schema feature rather than hand-replaying API calls.

## Pattern: Lookup Table

Simple reference data with a name and optional metadata.

```json
// Type
{"name": "application", "slug": "applications", "verbose_name": "Application", "verbose_name_plural": "Applications"}

// Fields
{"name": "app_name", "label": "Name", "type": "text", "required": true, "unique": true, "primary": true}
{"name": "owner_team", "label": "Owning Team", "type": "text"}                       // "owner" is reserved
{"name": "criticality", "label": "Criticality", "type": "select", "choice_set": <id>, "context": true}  // high/medium/low, shown when referenced
```

## Pattern: Relationship Bridge

Connect two core objects that don't have a built-in relationship.

```json
// Type: Links devices to applications
{"name": "device_application", "slug": "device-applications", "verbose_name": "Device Application Mapping",
 "display_expression": "{{ device }} → {{ application }}"}

// Fields
{"name": "device", "label": "Device", "type": "object", "app_label": "dcim", "model": "device", "required": true, "related_name": "application_mappings", "on_delete_behavior": "cascade"}
{"name": "application", "label": "Application", "type": "object", "app_label": "custom-objects", "model": "applications", "required": true, "related_name": "device_mappings", "on_delete_behavior": "protect"}
{"name": "role", "label": "Role", "type": "select", "choice_set": <id>}  // primary/secondary/dr
```

`display_expression` gives bridge rows a readable name without a synthetic primary field. `cascade` on the device side removes mappings when the device goes; `protect` on the application side stops an application from being deleted while mapped.

## Pattern: Extended Metadata

Structured metadata beyond what custom fields can model.

```json
// Type: DHCP Scopes tied to IP ranges
{"name": "dhcp_scope", "slug": "dhcp-scopes", "verbose_name": "DHCP Scope"}

// Fields
{"name": "scope_name", "label": "Name", "type": "text", "primary": true, "required": true, "unique": true}
{"name": "ip_range", "label": "IP Range", "type": "object", "app_label": "ipam", "model": "iprange", "required": true, "context": true}
{"name": "lease_time", "label": "Lease Time (seconds)", "type": "integer", "default": 86400, "validation_minimum": 300, "validation_maximum": 604800}
{"name": "dns_servers", "label": "DNS Servers", "type": "json"}
{"name": "is_active", "label": "Active", "type": "boolean", "default": true}
{"name": "runbook", "label": "Runbook", "type": "url"}   // v0.7+: instances can also set runbook_title
```

## Pattern: Hierarchical with Self-Reference

Model parent-child relationships within the same type.

```json
// Type
{"name": "cost_center", "slug": "cost-centers", "verbose_name": "Cost Center",
 "display_expression": "{{ code }}{% if description %} – {{ description }}{% endif %}"}

// Fields
{"name": "code", "label": "Code", "type": "text", "primary": true, "required": true, "unique": true}
{"name": "description", "label": "Description", "type": "longtext"}
{"name": "parent", "label": "Parent Cost Center", "type": "object", "app_label": "custom-objects", "model": "cost-centers", "related_name": "children", "on_delete_behavior": "protect"}
{"name": "budget", "label": "Annual Budget", "type": "decimal", "validation_minimum": 0}
```

`related_name: "children"` makes `cost_center.children.all()` available in export/config templates.

## Pattern: Multi-Object Collection

Track many-to-many relationships.

```json
// Type: Service catalog entries
{"name": "service_catalog", "slug": "service-catalog", "verbose_name": "Service Catalog Entry"}

// Fields
{"name": "service_name", "label": "Service", "type": "text", "primary": true, "required": true}
{"name": "devices", "label": "Hosting Devices", "type": "multiobject", "app_label": "dcim", "model": "device", "related_name": "services"}
{"name": "vms", "label": "Hosting VMs", "type": "multiobject", "app_label": "virtualization", "model": "virtualmachine", "related_name": "services"}
{"name": "owner_tenant", "label": "Owner", "type": "object", "app_label": "tenancy", "model": "tenant"}
```

Or collapse `devices` + `vms` into one polymorphic `multiobject` field (`hosts`) with `related_object_types_input` listing both models — one field, union in GraphQL, `?hosts_dcim_device=` / `?hosts_virtualization_virtualmachine=` filters in REST.

## Pattern: Config-Context-Aware Type (v0.6+)

Types whose instances should inherit ConfigContexts and feed config templates (NetBox 4.7+).

```json
// Type — config context can only be enabled at creation
{"name": "ospf_interface", "slug": "ospf-interfaces", "verbose_name": "OSPF Interface", "config_context_enabled": true}

// Fields — "site" and "platform" names + targets follow the aggregation convention
{"name": "name", "label": "Interface", "type": "text", "primary": true, "required": true}
{"name": "device", "label": "Device", "type": "object", "app_label": "dcim", "model": "device", "required": true, "related_name": "ospf_interfaces"}
{"name": "site", "label": "Site", "type": "object", "app_label": "dcim", "model": "site"}
{"name": "platform", "label": "Platform", "type": "object", "app_label": "dcim", "model": "platform"}
{"name": "area", "label": "Area", "type": "text", "required": true}
```

- Instances get `local_context_data`; site/platform contexts aggregate automatically; without any convention field, only local data applies.
- Config template (NetBox 4.7+): `{% for iface in custom_objects.ospf_interface.filter(device=device) %}` — by internal **name**.

## Pattern: Geo-Located Asset (v0.6+)

```json
{"name": "cell_site", "slug": "cell-sites", "verbose_name": "Cell Site"}

{"name": "site_code", "label": "Site Code", "type": "text", "primary": true, "required": true, "unique": true}
{"name": "position", "label": "Position", "type": "coordinates"}
{"name": "parent_site", "label": "NetBox Site", "type": "object", "app_label": "dcim", "model": "site"}
```

Write `position_latitude` and `position_longitude` together; the detail view shows a Map link.

## Display Names

Priority: `display_expression` (COT) → `primary` field → `<Verbose name> <id>`.

- Reference fields by **name** in the expression; undefined names render empty and render errors fall back silently, so a typo produces the primary-field name with no error via the API (the UI form checks Jinja syntax).
- Guard optional parts: `{{ name }}{% if site %} @ {{ site }}{% endif %}`.
- Keep it short — it is used in dropdowns, breadcrumbs, search results, and GraphQL `display`.

## Field Grouping

Use `group_name` to organize fields visually in forms:

```json
{"name": "hostname", "group_name": "Identity", "weight": 100}
{"name": "serial", "group_name": "Identity", "weight": 200}
{"name": "cpu_cores", "group_name": "Hardware", "weight": 100}
{"name": "ram_gb", "group_name": "Hardware", "weight": 200}
{"name": "site", "group_name": "Location", "weight": 100}
```

Fields within the same `group_name` are displayed together in the UI, ordered by `weight`.

## Navigation Grouping

Use `group_name` on the COT (not field) to organize types in the sidebar:

```json
{"name": "dhcp_scope", "group_name": "IPAM Extensions", ...}
{"name": "dns_zone", "group_name": "IPAM Extensions", ...}
{"name": "application", "group_name": "Service Catalog", ...}
```

Types with the same `group_name` appear together under a shared heading.

## Schema Lifecycle

- Set `version` on the COT (PEP 440) and bump it when fields change.
- Retire fields with `deprecated: true` + `deprecated_since` + `scheduled_removal` first; delete later and record a tombstone in `removed_fields` in the exported schema document.
- Promote schemas dev → staging → prod with `schema/preview/` then `schema/apply/` (see [api-workflow.md](api-workflow.md#portable-schema)) instead of re-issuing field POSTs by hand.
