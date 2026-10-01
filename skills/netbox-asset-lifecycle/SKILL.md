---
name: netbox-asset-lifecycle
description: >
  NetBox Asset Lifecycle — a commercial NetBox Labs feature that tracks equipment
  procurement from plan to install (BOMs, vendors, purchase orders, shipments,
  receiving) plus spares pools with stock-level allocations. Use when working with
  the asset-lifecycle REST or GraphQL API, driving greenfield procurement or
  brownfield sparing workflows over the API (generate BOM, install from shipment or
  spare, archive), or modeling vendors, purchase orders, shipments, and spares.
license: Apache-2.0
---

# NetBox Asset Lifecycle

NetBox Asset Lifecycle is a **commercial NetBox Labs feature** (NetBox Cloud / NetBox Enterprise) that tracks the procurement lifecycle of equipment — **plan → BOM → purchase order → shipment → install** — alongside a **spares** inventory for replacements. It builds on equipment you already model in NetBox DCIM: planned Racks/Devices/Modules/Cables roll up into a Bill of Materials, which is ordered, shipped, received, and installed, with traceability from each installed object back to the shipment or spares pool that fulfilled it.

> **Your knowledge of Asset Lifecycle may be outdated.** This is an early (0.x) feature — v0.3.0 was a **breaking** release that renamed models and REST endpoints, and every 0.2.x and earlier release was evaluation-only. Endpoints, fields, and status rules may still evolve. Prefer retrieval over pre-trained knowledge; verify the live API before generating code.

**Version note:** this skill targets plugin **v0.3.x** (v0.3.1, 2026-08-20) on **NetBox 4.5.4–4.7**. There is **no upgrade path from 0.2.x** — migrations were reinitialized in 0.3.0. See [Version Notes](#version-notes) for the 0.2→0.3 renames.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Asset Lifecycle docs | `https://netboxlabs.com/docs/asset-lifecycle/` | Features, per-model field reference, workflows, releases |
| Release notes | `.../asset-lifecycle/releases/` | Breaking changes, minimum NetBox version |
| NetBox DCIM models | `https://netboxlabs.com/docs/netbox/models/dcim/` | Core Device/Rack/Module/Cable + type models |
| Live API schema | `GET /api/schema/` (filter for `asset-lifecycle`) | Exact field names and action endpoints on the running version |
| NetBox MCP / Platform MCP | If configured — list/inspect asset-lifecycle endpoints | Verify live endpoints and field schemas |

## FIRST: Verify the Feature Is Available

Confirm the feature is enabled on the instance, your token has access, and you are on the 0.3 API surface:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" \
  "$NETBOX_URL/api/plugins/asset-lifecycle/assets/?limit=1" | python -m json.tool
```

- A paginated list (possibly empty) → feature enabled, **v0.3+** (`assets/` exists).
- **404** on `assets/` but **200** on `boms/` → a 0.2.x install; the `bom-objects/` endpoint and `spare_item` field in this skill's predecessor apply, and nothing below about allocations, archiving, or connector fields exists.
- **404** on `boms/` too → feature not enabled. **403** → token lacks asset-lifecycle object permissions.

---

## Scope

This skill covers the asset-lifecycle REST/GraphQL API and its procurement/sparing workflows. It does NOT cover:
- Modeling sites/devices/IPAM → use [netbox-data-modeling](../netbox-data-modeling/SKILL.md)
- Core REST/GraphQL patterns, auth, pagination, bulk → use [netbox-api-integration](../netbox-api-integration/SKILL.md)
- Bulk ingest of inventory → use [netbox-diode](../netbox-diode/SKILL.md)

**Prerequisite:** the equipment you procure must already exist in NetBox DCIM, typically as `status=planned` objects with a type assigned (Device `device_type`, Rack `rack_type`, Cable `type`). Asset Lifecycle does not create planned Devices/Racks — it tracks acquiring and installing them. (The one exception: [install a *new* object from a spare](#install-a-new-object-from-a-spare).)

## API Conventions

- **Base URL:** `/api/plugins/asset-lifecycle/`
- **Auth:** `Authorization: Bearer nbt_<key>.<secret>` (v2 token; v1 `Token <key>` still works until v5.0). Standard NetBox pagination (`?limit=`, `?offset=`), `?brief=true`, `?fields=`, `?q=` search, and `__` filter lookups all apply.
- **Object permissions:** standard `view_*`/`add_*`/`change_*`/`delete_*` per model (`netbox_asset_lifecycle.change_bom`, etc.). Action endpoints (`generate/`, `install/`) additionally require a token with **write enabled** and, for installs, the matching `dcim.change_*` / `dcim.add_*` permission on the target object.
- **Generic relations** (the equipment a line item / spare / allocation / asset points at) are written as **two fields**: a `*_type` ContentType (`{"app_label": "dcim", "model": "devicetype"}` or a pk) plus a `*_id` integer. The combined `item` / `assigned_object` field is read-only.
- **Nested FKs** accept the integer pk directly.
- **`variant`** is a JSON object and is never null *(0.3.1)* — send `{}` or omit it for "no variant". Blank attributes are dropped on save, so `{"airflow": ""}` is stored as `{}`.
- **Archived objects** are returned by default; filter with `?archived=false`.

## Quick Reference

| Endpoint (under `/api/plugins/asset-lifecycle/`) | Purpose |
|----------|---------|
| `boms/` · `POST boms/<id>/generate/` | BOM container; resolve scope rules into assets + line items |
| `bom-scope-rules/` | Include/exclude filter rules that define a BOM's scope |
| `bom-line-items/` | Type+variant+quantity rows (auto-generated or manual) |
| `assets/` · `POST assets/<id>/install/` | Per-instance planned object ↔ BOM junction; install from shipment or spare |
| `cable-types/` | Plugin-local cable type (`type`+`profile`+`connector_a`+`connector_b`) |
| `vendors/`, `vendor-accounts/` | Suppliers + accounts |
| `purchase-orders/` · `GET purchase-orders/<id>/export-line-items/?format=` | Orders; export lines as `json`/`csv`/`yaml` |
| `po-line-items/` | Ordered qty + price per BOM line (editable only while PO is `draft`) |
| `couriers/`, `courier-accounts/` | Shippers (UPS/FedEx/DHL seeded) + accounts |
| `shipments/`, `shipment-line-items/` | Deliveries + lines (`qty_shipped` / `qty_received`) |
| `spares-pools/`, `spare-items/` · `POST spare-items/<id>/install/` | Reserve inventory; create a new DCIM object from a spare |
| `spare-item-allocations/` | Min/max stock thresholds per pool + item + variant *(0.3)* |
| `status-transition-rules/` · `GET .../status-choices/?object_type_id=` | Editable FSM for BOM/PO/Shipment, incl. initial statuses |

GraphQL: every model is queryable alongside core types at `/graphql/` — `bom_list`, `purchase_order_list`, `shipment_list`, `asset_list`, `spare_item_list`, `spare_item_allocation_list`, etc. Type names carry an `AssetLifecycle` prefix *(0.3.1)*: `AssetLifecycleBOM`, `AssetLifecyclePurchaseOrder`, `AssetLifecycleShipment`, `AssetLifecycleAsset`, `AssetLifecycleSpareItem`, …

## Data Model

See [references/data-model.md](references/data-model.md) for the full object graph, every model's fields, and uniqueness constraints. Load it when building requests or debugging validation errors.

```
Vendor ─ VendorAccount
   │
BOM ─ BOMScopeRule / BOMLineItem / Asset       (BOM rolls planned DCIM objects up into line items;
   │                                             one Asset per matched planned object)
PurchaseOrder ─ POLineItem                     (PO requires a BOM + Vendor)
   │
Shipment ─ ShipmentLineItem                    (Shipment requires a PO + Courier + tracking_number)
   │
Asset.shipment  ◄── install from shipment      (records which delivery fulfilled the planned object)
Asset.spares_pool ◄── install from spare       (records which pool it was drawn from)

SparesPool ─ SpareItem / SpareItemAllocation   (reserve inventory + min/max stock thresholds)
```

`BOM`, `PurchaseOrder`, and `Shipment` carry a rule-governed `status` and an `archived` flag.

## Status & Transition Rules

A status change on a BOM, PO, or Shipment is **only allowed if a matching `StatusTransitionRule` exists**; an illegal jump returns **400** (`Cannot transition from X to Y`). A rule with a **blank `from_status` is an initial-status rule** — it declares a status an object may be *created* with; creating with any other status returns 400 (`… is not a valid initial status`). `SpareItem.status` is free-form (not rule-governed).

| Model | Status values | Default initial |
|-------|---------------|-----------------|
| BOM | `draft`, `approved`, `ordered`, `fulfilled`, `cancelled` | `draft` |
| PurchaseOrder | `draft`, `approved`, `ordered`, `fulfilled`, `cancelled` | `draft` |
| Shipment | `prepared`, `shipped`, `received`, `cancelled`, `returned`, `lost` | `prepared` **or** `shipped` *(0.3)* |
| SpareItem (free) | `serviceable`, `damaged`, `missing` | any |

Default rules: BOM/PO move `draft ↔ approved ↔ ordered ↔ fulfilled` (plus `approved ↔ fulfilled`), cancellation one-way from `approved`/`ordered` only. Shipments move `prepared ↔ shipped ↔ received`, `shipped ↔ lost`, `lost ↔ received`, `received ↔ returned`, cancellation one-way from `prepared`/`shipped`. There is **no** default `draft → ordered`, `draft → fulfilled`, or `draft → cancelled`.

```bash
# Discover valid status strings for an object type
GET status-transition-rules/status-choices/?object_type_id=<contenttype_id>

# Allow an otherwise-illegal jump by adding a rule first
POST status-transition-rules/
{"object_type": {"app_label":"netbox_asset_lifecycle","model":"purchaseorder"},
 "from_status": "draft", "to_status": "ordered", "allow_reverse": false}

# Allow shipments to be created directly as "received" (initial-status rule)
POST status-transition-rules/
{"object_type": {"app_label":"netbox_asset_lifecycle","model":"shipment"},
 "from_status": "", "to_status": "received"}
```

See [references/status-transitions.md](references/status-transitions.md) for the complete default rule sets and the model-specific locks (BOM "not current", PO `order_id`, received-shipment site, archived). Load it when a status change returns 400.

## Action Endpoints (the steps that used to be UI-only)

Generation, install, and export have REST endpoints since 0.2.0; receiving is plain field editing. **Use these instead of reconstructing their effects by hand** — they enforce the integrity the feature exists for (rollup correctness, spare decrement, shipment→object traceability).

| Action | Request | Requires | Returns |
|--------|---------|----------|---------|
| Generate BOM | `POST boms/<id>/generate/` (empty body) | `change_bom`; BOM `draft` with ≥1 enabled scope rule | The BOM (`last_generated`, `is_current=true`); 400 otherwise |
| Install from shipment | `POST assets/<id>/install/ {"shipment": <id>}` | `change_asset` + `dcim.change_<model>`; shipment `received` and on this BOM's PO; asset not yet installed | The Asset with `shipment` + `installed` set |
| Install from spare | `POST assets/<id>/install/ {"spare_item": <id>}` | as above; spare `serviceable`, `quantity ≥ 1`, exact `item_type`/`item_id`/`variant` match | The Asset with `spares_pool` + `installed` set; spare `quantity` decremented; 409 if drained concurrently |
| New object from spare | `POST spare-items/<id>/install/ {<DCIM object fields>}` | `change_spareitem` + `dcim.add_<model>`; spare `serviceable`, `quantity ≥ 1`; device/rack/module types only | 201 + the new DCIM object; spare decremented |
| Export PO lines | `GET purchase-orders/<id>/export-line-items/?format=json` | `view_purchaseorder` | File body (`json` default; `csv`, `yaml`) |

**Install over REST does not touch the DCIM object** — it links the asset and (for spares) consumes inventory. Follow it with a normal `PATCH /api/dcim/devices/<id>/ {"status": "staged", "serial": "..."}` to update the object itself; the UI install form does both in one submit.

**Receiving** is field editing: `PATCH shipments/<id>/ {"status": "received", "site": <id>, "date_received": "..."}` (a `site` and/or `location` is **required** to enter `received`), then `PATCH shipment-line-items/<id>/ {"qty_received": N}` per line — a `qty_received` below `qty_shipped` is the documented way to record shortfall/damage. The UI's **Mark Received** button is just a bulk form for the same writes.

## Greenfield Procurement Workflow

Full step-by-step with payloads and validation notes in [references/workflow-greenfield.md](references/workflow-greenfield.md).

1. **Vendor (+ optional account)**
   ```bash
   POST vendors/          {"name": "Acme Networks", "code": "ACME"}
   POST vendor-accounts/  {"vendor": <vendor_id>, "account_number": "ACC-001"}
   ```
2. **BOM + scope rules → generate → approve.** Author scope rules, not line items; scope rules and line items are locked once the BOM leaves `draft`.
   ```bash
   POST boms/             {"name": "DC1 Spine Refresh"}                       # status defaults to draft
   POST bom-scope-rules/  {"bom": <bom_id>, "action": "include",
                           "object_types": [{"app_label":"dcim","model":"device"}],
                           "parameters": {"site_id":[3], "role_id":[5], "status":["planned"]}}
   POST boms/<bom_id>/generate/
   GET  bom-line-items/?bom_id=<bom_id>                                       # collect line item ids
   PATCH boms/<bom_id>/   {"status": "approved"}
   ```
3. **Purchase order + PO line items** (lines editable only while `draft`; `order_id` required to reach `ordered`/`fulfilled`; `currency` falls back to the `default_currency` plugin setting when omitted):
   ```bash
   POST purchase-orders/  {"vendor": <vendor_id>, "vendor_account": <acct_id>, "bom": <bom_id>, "currency": "USD"}
   POST po-line-items/    {"purchase_order": <po_id>, "bom_line_item": <bli_id>, "qty_ordered": 16, "unit_price": "1200.00"}
   PATCH purchase-orders/<po_id>/  {"status": "approved"}
   PATCH purchase-orders/<po_id>/  {"order_id": "PO-7788", "status": "ordered"}
   ```
4. **Shipment + shipment line items** (create as `prepared` or `shipped`; UPS/FedEx/DHL seeded):
   ```bash
   POST shipments/        {"purchase_order": <po_id>, "courier": <courier_id>, "tracking_number": "1Z...",
                           "status": "shipped", "site": <site_id>, "date_shipped": "2026-09-01", "date_expected": "2026-09-05"}
   POST shipment-line-items/  {"shipment": <ship_id>, "bom_line_item": <bli_id>, "qty_shipped": 16}
   ```
5. **Receive:** `PATCH shipments/<ship_id>/ {"status": "received", "date_received": "..."}` then `PATCH shipment-line-items/<id>/ {"qty_received": 16}`.
6. **Install:** for each asset on the BOM (`GET assets/?bom_id=<bom_id>&installed__isnull=true`), `POST assets/<id>/install/ {"shipment": <ship_id>}`, then PATCH the DCIM object's own `status`/`serial`.
7. **Close out:** PO and BOM → `fulfilled`; optionally `PATCH … {"archived": true}` on shipments, POs, and the BOM.

## Brownfield Sparing Workflow

Full detail in [references/workflow-sparing.md](references/workflow-sparing.md).

```bash
POST spares-pools/  {"name": "HQ Cold Storage", "site": <site_id>, "location": <loc_id>}
POST spare-items/   {"pool": <pool_id>, "item_type": {"app_label":"dcim","model":"devicetype"},
                     "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                     "quantity": 5, "status": "serviceable"}
# Target stock level for that item in that pool (advisory; surfaces low/excess stock)
POST spare-item-allocations/  {"pool": <pool_id>, "item_type": {"app_label":"dcim","model":"devicetype"},
                     "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                     "min_quantity": 2, "max_quantity": 8}
GET  spare-item-allocations/?pool_id=<pool_id>     # read-only current_quantity / below_minimum / above_maximum / is_fulfilled
```

- Set `status` to `damaged`/`missing` to flag inventory-audit discrepancies (free-form).
- Serialized single units set `serial`/`asset_tag` with `quantity=1` (quantity >1 with a serial/asset_tag is rejected).
- **Replace a failed planned unit from stock:** `POST assets/<id>/install/ {"spare_item": <id>}` — decrements the spare (to 0, record retained) and records the **pool** on the asset. Then PATCH the DCIM object's status.
- **Stand up a brand-new object from a spare** (no BOM involved): `POST spare-items/<id>/install/ {"name": "...", "site": <id>, "role": <id>, "status": "active"}` — creates the Device/Rack/Module from the spare's type, serial, asset tag, and variant. Cables cannot be installed this way.
- Allocations only count `serviceable` items with the exact same `variant`; a pool holds one allocation per item+variant.

## Archiving *(0.3)*

`BOM`, `PurchaseOrder`, and `Shipment` have a writable `archived` boolean. Archive when the object is done (typically `fulfilled`/`cancelled`/`received`) to hide it from list views without deleting it: `PATCH boms/<id>/ {"archived": true}`; reverse with `false`. Rules:
- An object **cannot be created** as `archived: true` — create, then archive.
- While archived, its **child objects reject writes** (scope rules, line items) and the UI makes the object read-only. Unarchive before editing.
- Lists return archived objects by default over REST; use `?archived=false` for the UI's default view.

## Anti-Patterns

1. **Using 0.2-era paths/fields.** `bom-objects/` → `assets/`; `spare_item` on an asset → `spares_pool`. Both return 404/400 on 0.3.
2. **Hand-authoring line items to replace generation.** Author scope rules and `POST boms/<id>/generate/`. Manual line items are for one-offs only; auto-generated rows are immutable and rebuilt on every generate.
3. **Faking install.** Never PATCH `assets/<id>/ {"shipment": …}` or decrement `spare-items/<id>/ quantity` by hand — use `assets/<id>/install/`, which validates the shipment/spare, consumes inventory atomically, and stamps `installed`. Never PATCH a DCIM object to `active` *instead of* installing; do it *after*.
4. **Skipping required parents.** A PO requires `bom` + `vendor`; a Shipment requires `purchase_order` + `courier` + `tracking_number`; line items require their parent + a `bom_line_item`.
5. **Illegal status jumps or initial statuses.** `draft→ordered`, creating a shipment as `received`, etc. return 400 by default. Use `status-choices/` to discover values and `status-transition-rules/` to permit new transitions or initial statuses.
6. **Ordering before the PO is `ordered`.** Set `order_id` and move the PO to `ordered` before creating shipments — the UI only offers *Create Shipment* on ordered POs.
7. **Editing locked children.** Scope rules and BOM line items are writable only while the BOM is `draft`; PO line items only while the PO is `draft`; all children are locked while the parent is archived. Revert status (if a rule permits) or unarchive first.
8. **`received` without a destination.** A shipment cannot enter `received` without a `site` and/or `location` → 400.
9. **Mismatched accounts/locations/dates.** `vendor_account.vendor` must equal the PO `vendor`; `courier_account.courier` must equal the Shipment `courier`; `location.site` must equal `site`; `date_expected`/`date_received` cannot precede `date_shipped`.
10. **Uniqueness traps.** BOM `name`; Vendor/Courier `name` (+ `code` if set); `(vendor, account_number)`; `(vendor, order_id)` when set; `(courier, tracking_number)`; `(purchase_order, bom_line_item)`; SparesPool `name`; SpareItem `asset_tag` (global) and `(item_type, item_id, serial)`; non-serialized SpareItem `(pool, item_type, item_id, variant, status)`; Allocation `(pool, item_type, item_id, variant)`; CableType `(type, profile, connector_a, connector_b)`.
11. **Wrong content type for a generic relation.** Line items, spares, and allocations point at **type** models (`dcim.devicetype`/`moduletype`/`racktype` or `netbox_asset_lifecycle.cabletype`); assets point at **instance** models (`dcim.device`/`rack`/`module`/`cable`).
12. **Variant mismatch on install.** A spare fulfils an asset only if `item_type`, `item_id`, and `variant` match **exactly** after normalization. Stock a `{"airflow":"front-to-rear"}` spare for a planned device whose airflow is set; `{}` will not match it.
13. **Assuming receipt activates inventory.** It doesn't — install is a separate step that links the asset; the DCIM object's status is yours to update.

## Version Notes

### Plugin 0.3.1 (2026-08-20) — NetBox 4.5.4–4.7
- BOM shown on the shipment detail view.
- GraphQL types/enums prefixed `AssetLifecycle…` (breaking for existing GraphQL queries).
- `variant` can no longer be null on BOM line items, spare items, and allocations.

### Plugin 0.3.0 (2026-06-29) — NetBox 4.5.4–4.6 — **breaking**
- Migrations reinitialized: **no upgrade from 0.2.x**; earlier releases were evaluation-only.
- `BOMObject` → `Asset`; REST `bom-objects/` → `assets/`.
- `Asset.spare_item` → `Asset.spares_pool` (the *pool*, not the consumed unit, is recorded).
- New: `SpareItemAllocation` (`spare-item-allocations/`, min/max thresholds); `default_currency` plugin setting; `connector_a`/`connector_b` on `CableType`; multiple initial shipment statuses (`prepared` or `shipped`); `archived` on BOM/PO/Shipment.

### Plugin 0.2.x (2026-06) — NetBox 4.5.3–4.6 (evaluation only)
- REST actions `boms/<id>/generate/`, `assets/<id>/install/` (as `bom-objects/`), `spare-items/<id>/install/`; GraphQL API; PO line item export; spares-first install; scope rules locked on non-draft BOMs; spare retained at quantity 0.

## References

- [references/data-model.md](references/data-model.md) — Full object graph, per-model fields, constraints, filters. Load when building requests or decoding validation errors.
- [references/status-transitions.md](references/status-transitions.md) — Default transition and initial-status rule sets + every model-specific 400 lock. Load on a status 400.
- [references/workflow-greenfield.md](references/workflow-greenfield.md) — Complete procurement walkthrough with payloads: generate, order, ship, receive, install, archive.
- [references/workflow-sparing.md](references/workflow-sparing.md) — Spares pools, allocations and stock alerts, audits, install-from-spare, new-object-from-spare.
