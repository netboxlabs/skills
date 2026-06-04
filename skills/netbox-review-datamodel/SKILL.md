---
name: netbox-review-datamodel
description: >
  Audit and review NetBox data model design choices. Use when evaluating site hierarchy,
  IPAM organization, device modeling, tenancy patterns, custom field vs tag decisions,
  naming conventions, and extensibility strategy. Identifies modeling anti-patterns and
  suggests improvements aligned with NetBox best practices.
license: Apache-2.0
---

# NetBox Data Model Review

> **Your knowledge of NetBox data models may be outdated.** Available model types, relationships, and extensibility mechanisms evolve between releases. Verify against current docs before recommending changes.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Data model docs | `https://netboxlabs.com/docs/netbox/models/` | All available model types |
| Custom fields docs | `https://netboxlabs.com/docs/netbox/customization/custom-fields/` | Field types and behavior |
| Custom Objects docs | `https://netboxlabs.com/docs/extensions/custom-objects/` | No-code extensibility |
| NetBox MCP server | If configured — inspect the current data model, count objects, find inconsistencies | Live model audit |

## Review Workflow

Follow this checklist when auditing a NetBox data model:

1. **Understand the organization** — what kind of network? (enterprise, DC, ISP, hybrid)
2. **Check hierarchy** — is the Region → Site Group → Site → Location → Rack hierarchy appropriate?
3. **Check IPAM** — are prefixes properly nested? VRFs used where needed? Aggregates defined?
4. **Check naming** — are slugs consistent? Are names human-readable and grep-friendly?
5. **Check extensibility choices** — custom fields vs tags vs config contexts vs custom objects
6. **Check tenancy** — is multi-tenancy used correctly? Over-tenanted or under-tenanted?
7. **Check for modeling smell** — overloaded fields, misuse of description for structured data

## Critical Rules

### Hierarchy (HIER)

| ID | Rule | Severity |
|----|------|----------|
| HIER-1 | Use Regions for geography, Site Groups for function — don't conflate | High |
| HIER-2 | Locations are recursive — don't flatten (Floor > Room > Row is better than Row alone) | Medium |
| HIER-3 | Every device should be in a Site (never orphaned at top level) | High |
| HIER-4 | Racks should have a Location when the site has locations defined | Medium |
| HIER-5 | Don't create single-child hierarchies — they add complexity without value | Low |
| HIER-6 | On 4.6+, **RackGroups** offer a flat, cross-location way to group racks (e.g. "cage-7", "cold-aisle-B") independent of the Location tree — use them when rack grouping doesn't map cleanly onto site/location nesting, not as a substitute for Locations | Low |

### IPAM (IPAM)

| ID | Rule | Severity |
|----|------|----------|
| IPAM-1 | Define RIR Aggregates to track address space allocation boundaries | Medium |
| IPAM-2 | Prefixes should nest correctly (child within parent CIDR) | High |
| IPAM-3 | Use VRFs to separate overlapping address spaces — never duplicate prefixes in global table | Critical |
| IPAM-4 | IP Addresses must have prefix length (`/32` for loopbacks, actual mask for interfaces) | High |
| IPAM-5 | Use Roles to classify prefix purpose (infrastructure, customer, management); on NetBox 4.6+ **ASNs can also carry a Role** — use it to classify ASN purpose | Medium |
| IPAM-6 | VLANs should be in VLAN Groups with an appropriate **scope**. Scope can be Region/SiteGroup/Site/Location/Rack/ClusterGroup/Cluster (and **RackGroup** on 4.6+) — don't assume site/location are the only options; pick the tightest scope that matches the VLANs' reuse boundary | Medium |

### Device Modeling (DEV)

| ID | Rule | Severity |
|----|------|----------|
| DEV-1 | Every device needs a Device Type (not just a name) — this enables port planning | High |
| DEV-2 | Use Roles to classify function (router, switch, firewall) — not naming convention alone | High |
| DEV-3 | Platforms indicate software — assign them for config template compatibility | Medium |
| DEV-4 | Virtual chassis members should have proper VC position and master assignment | Medium |
| DEV-5 | Interface types must match reality (1000base-t vs 10gbase-sr) for capacity planning | Medium |

### Extensibility (EXT)

| ID | Rule | Severity |
|----|------|----------|
| EXT-1 | Use **custom fields** for single-valued structured data attached to one object type | — |
| EXT-2 | Use **tags** for cross-object-type classification and boolean "has this property" | — |
| EXT-3 | Use **config contexts** for hierarchical key-value data that merges by scope | — |
| EXT-4 | Use **custom objects** when you need a new first-class entity with its own relationships | — |
| EXT-5 | Never store structured data (JSON, lists) in description or comments fields | High |
| EXT-6 | Don't create custom fields that duplicate built-in fields (e.g., custom "location" field) | High |
| EXT-7 | Prefer custom objects over dozens of custom fields when the data is really a related entity | Medium |
| EXT-8 | On 4.6+, attach a **`validation_schema`** (JSON Schema) to JSON custom fields to enforce structure instead of leaving them free-form; flag JSON fields holding structured data with no schema | Medium |

### Naming (NAME)

| ID | Rule | Severity |
|----|------|----------|
| NAME-1 | Slugs should be lowercase, hyphenated, grep-friendly (`nyc-dc1` not `NYC_DC1`) | Medium |
| NAME-2 | Device names should encode location + function + index (`sw-nyc-dc1-01`) | Low |
| NAME-3 | Be consistent — pick one naming scheme and apply it everywhere | High |
| NAME-4 | Avoid embedding metadata in names (don't put VLAN ID in site name) | Medium |

### Tenancy (TEN)

| ID | Rule | Severity |
|----|------|----------|
| TEN-1 | Use tenants for logical ownership boundaries (customer, department, project) | — |
| TEN-2 | Don't over-tenant — if everything belongs to one tenant, you probably don't need tenancy | Medium |
| TEN-3 | Tenant Groups organize tenants hierarchically (e.g., by business unit) | Low |
| TEN-4 | Shared infrastructure (management networks, core routers) can be untenanted | Medium |

## Anti-Patterns to Flag

| Anti-Pattern | Why It Matters | Fix |
|-------------|---------------|-----|
| Flat site hierarchy (no locations) | Can't track floor/room/row placement | Add Location hierarchy |
| All IPs in global VRF with duplicates | IPAM conflicts, broken reports | Separate into VRFs |
| Custom fields used as foreign keys | No referential integrity, hard to query | Use custom objects or tags |
| JSON blobs in description fields | Not searchable, not validated, not filterable | Use custom fields or config contexts |
| One giant "catch-all" custom field per type | Defeats the purpose of structured data | Split into individual fields |
| Devices without Device Types | Loses port/bay/slot modeling | Always specify hardware model |
| Mixing naming conventions | Impossible to script against, confusing | Standardize and bulk-rename |
| Tags with spaces or special characters | Breaks API filtering, scripts | Use slug-friendly tag names |
| Over-nested regions (> 3 levels) | Adds complexity without improving navigation | Flatten to 2-3 levels max |
| Prefix hierarchy gaps | Parent/child relationships broken | Fill missing intermediate prefixes |

## Scope

This skill covers **data model design and organization**. It does NOT cover:
- Code that interacts with the API → use [netbox-review-integration](../netbox-review-integration/SKILL.md)
- How to model from scratch → use [netbox-data-modeling](../netbox-data-modeling/SKILL.md)
- Custom Objects plugin specifics → use [netbox-custom-objects](../netbox-custom-objects/SKILL.md)

## Principles

- **Context matters.** A 5-site enterprise has different needs than a 500-site ISP. Don't over-engineer.
- **Audit the real instance.** If MCP is available, query actual data rather than reviewing documentation alone.
- **Suggest, don't dictate.** Many modeling choices are valid — flag clear anti-patterns, but acknowledge trade-offs for judgment calls.
- **Prioritize data integrity.** Issues that cause broken relationships or duplicate data are more important than cosmetic naming issues.
- **Consider migration cost.** Flagging an anti-pattern is more useful if you also explain how hard it is to fix with existing data.
