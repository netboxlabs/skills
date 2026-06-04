# Modeling Patterns

Common patterns for designing custom object types.

## Before Creating a Custom Object

Ask these questions first:

1. **Does a built-in NetBox model already cover this?** Check DCIM, IPAM, Circuits, Tenancy, Virtualization first.
2. **Would custom fields on an existing model suffice?** If you just need 2-3 extra fields on Device or Site, custom fields are simpler.
3. **Do you need complex business logic?** Custom objects are data-only — no custom validation beyond field-level rules. For complex workflows, consider a plugin.

## Pattern: Lookup Table

Simple reference data with a name and optional metadata.

```json
// Type
{"name": "application", "slug": "applications", "verbose_name": "Application", "verbose_name_plural": "Applications"}

// Fields
{"name": "app_name", "label": "Name", "type": "text", "required": true, "unique": true, "primary": true}
{"name": "owner", "label": "Owner", "type": "text"}
{"name": "criticality", "label": "Criticality", "type": "select", "choice_set": <id>}  // high/medium/low
```

## Pattern: Relationship Bridge

Connect two core objects that don't have a built-in relationship.

```json
// Type: Links devices to applications
{"name": "device_application", "slug": "device-applications", "verbose_name": "Device Application Mapping"}

// Fields
{"name": "device", "label": "Device", "type": "object", "app_label": "dcim", "model": "device", "required": true}
{"name": "application", "label": "Application", "type": "object", "app_label": "custom-objects", "model": "applications", "required": true}  // v0.5+: model = target COT slug
{"name": "role", "label": "Role", "type": "select", "choice_set": <id>}  // primary/secondary/dr
```

## Pattern: Extended Metadata

Add structured metadata that goes beyond what custom fields can model.

```json
// Type: DHCP Scopes tied to IP ranges
{"name": "dhcp_scope", "slug": "dhcp-scopes", "verbose_name": "DHCP Scope"}

// Fields
{"name": "scope_name", "label": "Name", "type": "text", "primary": true, "required": true, "unique": true}
{"name": "ip_range", "label": "IP Range", "type": "object", "app_label": "ipam", "model": "iprange", "required": true}
{"name": "lease_time", "label": "Lease Time (seconds)", "type": "integer", "default": 86400, "validation_minimum": 300, "validation_maximum": 604800}
{"name": "dns_servers", "label": "DNS Servers", "type": "json"}
{"name": "is_active", "label": "Active", "type": "boolean", "default": true}
```

## Pattern: Hierarchical with Self-Reference

Model parent-child relationships within the same type.

```json
// Type
{"name": "cost_center", "slug": "cost-centers", "verbose_name": "Cost Center"}

// Fields
{"name": "code", "label": "Code", "type": "text", "primary": true, "required": true, "unique": true}
{"name": "description", "label": "Description", "type": "longtext"}
{"name": "parent", "label": "Parent Cost Center", "type": "object", "app_label": "custom-objects", "model": "cost-centers"}  // v0.5+: self-ref uses the COT's own slug
{"name": "budget", "label": "Annual Budget", "type": "decimal", "validation_minimum": 0}
```

## Pattern: Multi-Object Collection

Track many-to-many relationships.

```json
// Type: Service catalog entries
{"name": "service_catalog", "slug": "service-catalog", "verbose_name": "Service Catalog Entry"}

// Fields
{"name": "service_name", "label": "Service", "type": "text", "primary": true, "required": true}
{"name": "devices", "label": "Hosting Devices", "type": "multiobject", "app_label": "dcim", "model": "device"}
{"name": "vms", "label": "Hosting VMs", "type": "multiobject", "app_label": "virtualization", "model": "virtualmachine"}
{"name": "owner_tenant", "label": "Owner", "type": "object", "app_label": "tenancy", "model": "tenant"}
```

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
