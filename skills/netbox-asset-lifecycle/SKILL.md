---
name: netbox-asset-lifecycle
description: >
  NetBox Asset Lifecycle — a commercial NetBox Labs feature that tracks equipment
  procurement from plan to install (BOMs, vendors, purchase orders, shipments,
  receiving) plus spares management. Use when working with the asset-lifecycle REST
  API, driving greenfield procurement or brownfield sparing workflows, or modeling
  vendors, purchase orders, shipments, and spares pools. Generation, receiving, and
  install are UI-only — the skill explains how to drive them or hand off.
license: Apache-2.0
---

# NetBox Asset Lifecycle

NetBox Asset Lifecycle is a **commercial NetBox Labs feature** (NetBox Cloud / NetBox Enterprise) that tracks the procurement lifecycle of equipment — **plan → BOM → purchase order → shipment → install** — alongside a **spares** inventory for replacements. It builds on equipment you already model in NetBox DCIM: planned Racks/Devices/Modules/Cables roll up into a Bill of Materials, which is ordered, shipped, received, and installed, with traceability back to the shipment or spare that fulfilled each object.

> **Your knowledge of Asset Lifecycle may be outdated.** This is a Public Preview feature — endpoints, fields, and status rules may evolve between releases. Prefer retrieval over pre-trained knowledge; verify the live API before generating code.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Asset Lifecycle docs | `https://netboxlabs.com/docs/developer/plugins-extensions/asset-lifecycle/` | Feature reference, models, workflows |
| NetBox DCIM models | `https://netboxlabs.com/docs/netbox/models/dcim/` | Core Device/Rack/Module/Cable + type models |
| NetBox MCP / Platform MCP | If configured — list/inspect asset-lifecycle endpoints | Verify live endpoints and field schemas |

## FIRST: Verify the Feature Is Available

Confirm the feature is enabled on the instance and your token has access:

```bash
curl -s -H "Authorization: Token $NETBOX_TOKEN" \
  "$NETBOX_URL/api/plugins/asset-lifecycle/boms/" | python -m json.tool
```

A paginated list (possibly empty) means it's available. **404** = feature not enabled. **403** = your token lacks the asset-lifecycle object permissions.

---

## Scope

This skill covers the asset-lifecycle REST API and its procurement/sparing workflows. It does NOT cover:
- Modeling sites/devices/IPAM → use [netbox-data-modeling](../netbox-data-modeling/SKILL.md)
- Core REST/GraphQL patterns, auth, pagination → use [netbox-api-integration](../netbox-api-integration/SKILL.md)
- Bulk ingest of inventory → use [netbox-diode](../netbox-diode/SKILL.md)

**Prerequisite:** the equipment you procure must already exist in NetBox DCIM, typically as `status=planned` objects. Asset Lifecycle does not create Devices/Racks — it tracks acquiring and installing them.

## API Conventions

- **Base URL:** `/api/plugins/asset-lifecycle/`
- **Auth:** `Authorization: Token <key>`. Standard NetBox pagination (`?limit=`, `?offset=`), `?brief=true`, `?fields=`, `?q=` search, and `__` filter lookups all apply.
- **Object permissions:** standard `add_*`/`change_*`/`delete_*`/`view_*` per model.
- **Generic relations** (the equipment a line item / spare / BOM object points at) are written as **two fields**: a `*_type` ContentType (as `{"app_label": "dcim", "model": "devicetype"}` or a pk) plus a `*_id` integer. The combined `item` / `assigned_object` field is read-only.
- **Nested FKs** accept the integer pk directly.

## UI-Only Operations — Do Not Fake Them Over the API

In the current Public Preview, three operations have **no REST endpoint** and are driven through the NetBox web UI:

| UI-only operation | Where it lives in the UI |
|-------------------|--------------------------|
| **BOM generation** — resolve a BOM's scope rules into BOM objects + rolled-up line items | On a draft BOM's detail page → **Generate** |
| **Receiving** — record receipt against a shipment ("Mark Received") | On a shipment's detail page → **Mark Received** |
| **Install** — activate the planned DCIM object and link it to the shipment/spare that fulfilled it | On a planned object's detail page → **Install → Install from shipment / from spares pool** |

**Do not reconstruct these over the API.** Never hand-author BOM line items to substitute for generation, never PATCH `qty_received` to fake receiving, and never PATCH DCIM objects to fake an install. These shortcuts lose the integrity the feature is designed to enforce (auto-rollup correctness, receive accounting, install traceability back to a shipment or spare).

Instead, for any UI-only step:

- **If you have browser automation or computer-use capabilities**, drive the NetBox UI directly to complete the step, then resume the API-driven work.
- **Otherwise, hand off to the user.** Tell them exactly which UI action to take (e.g. "Open BOM *DC1 Spine Refresh* and click **Generate**"), then continue once they confirm it's done.

Everything else in the procurement chain — vendors, accounts, the BOM container, scope rules, purchase orders, shipments, spares, and status transitions — is fully API-driven and is the agent's job.

## Data Model

See [references/data-model.md](references/data-model.md) for the full object graph, every model's fields, and uniqueness constraints. Load it when building requests or debugging validation errors.

The chain at a glance:

```
Vendor ─ VendorAccount
   │
BOM ─ BOMScopeRule / BOMLineItem / BOMObject       (BOM rolls up planned DCIM objects)
   │
PurchaseOrder ─ POLineItem          (PO requires a BOM + Vendor)
   │
Shipment ─ ShipmentLineItem         (Shipment requires a PO + Courier + tracking_number)
   │
(install) → activates the planned DCIM object

SparesPool ─ SpareItem              (alternate fulfillment path for replacements)
```

Stateful models (`BOM`, `PurchaseOrder`, `Shipment`) are governed by an editable transition-rule system — see below.

## Status & Transition Rules

Three models carry a status governed by **StatusTransitionRule** records. A status change is **only allowed if a matching rule exists**; an illegal jump returns **400**. `SpareItem.status` is free-form (not rule-governed).

| Model | Status values |
|-------|---------------|
| BOM | `draft`, `approved`, `ordered`, `fulfilled`, `cancelled` |
| PurchaseOrder | `draft`, `approved`, `ordered`, `fulfilled`, `cancelled` |
| Shipment | `prepared`, `shipped`, `received`, `cancelled`, `returned`, `lost` |
| SpareItem (free) | `serviceable`, `damaged`, `missing` |

Default rules (shipped with the feature): BOM/PO move `draft→approved→ordered→fulfilled`, with cancellation only from `approved`/`ordered`. Shipments move `prepared→shipped→received`, cancellation only while `prepared`/`shipped`. There is **no** default `draft→ordered`, `draft→fulfilled`, or `draft→cancelled`.

Discover valid targets and add new transitions via the API:

```bash
# Discover valid status strings for an object type
GET /api/plugins/asset-lifecycle/status-transition-rules/status-choices/?object_type_id=<id>

# Allow an otherwise-illegal jump by adding a rule first
POST /api/plugins/asset-lifecycle/status-transition-rules/
{"object_type": {"app_label":"netbox_asset_lifecycle","model":"purchaseorder"},
 "from_status": "draft", "to_status": "ordered", "allow_reverse": false}
```

See [references/status-transitions.md](references/status-transitions.md) for the complete default rule sets and the BOM "not current" lock. Load it when a status change returns 400.

## Greenfield Procurement Workflow

Full step-by-step with payloads in [references/workflow-greenfield.md](references/workflow-greenfield.md). API-driven steps are the agent's job; the **Generate**, **Receive**, and **Install** steps are UI-only (drive the UI if equipped, else hand off — see [UI-Only Operations](#ui-only-operations--do-not-fake-them-over-the-api)).

1. **Vendor (+ optional account)** — *API*
   ```bash
   POST vendors/          {"name": "Acme Networks", "code": "ACME"}
   POST vendor-accounts/  {"vendor": <vendor_id>, "account_number": "ACC-001"}
   ```
2. **BOM container + scope rules** — *API*. Create the BOM and add scope rules that describe which planned DCIM objects it covers; do **not** hand-author line items.
   ```bash
   POST boms/             {"name": "DC1 Spine Refresh", "status": "draft"}
   POST bom-scope-rules/  {"bom": <bom_id>, "action": "include",
                           "object_types": [{"app_label":"dcim","model":"device"}],
                           "parameters": {"site_id":[3], "role_id":[5], "status":["planned"]}}
   ```
3. **Generate the BOM** — *UI-only*. On the draft BOM → **Generate** to resolve scope rules into BOM objects + rolled-up line items. Then approve over the API once it looks right:
   ```bash
   PATCH boms/<bom_id>/   {"status": "approved"}
   ```
4. **Purchase order + PO line items** — *API* (PO requires a BOM; non-draft requires `order_id`). Reference the generated `bom_line_item` ids:
   ```bash
   POST purchase-orders/  {"vendor": <vendor_id>, "vendor_account": <acct_id>,
                           "bom": <bom_id>, "status": "draft", "currency": "USD"}
   POST po-line-items/    {"purchase_order": <po_id>, "bom_line_item": <bli_id>,
                           "qty_ordered": 16, "unit_price": "1200.00"}
   PATCH purchase-orders/<po_id>/  {"order_id": "PO-7788", "status": "approved"}
   ```
5. **Shipment + shipment line items** — *API* (UPS/FedEx/DHL couriers are pre-seeded):
   ```bash
   POST shipments/        {"purchase_order": <po_id>, "courier": <courier_id>,
                           "tracking_number": "1Z...", "status": "prepared",
                           "site": <site_id>, "date_expected": "2026-06-05"}
   POST shipment-line-items/  {"shipment": <ship_id>, "bom_line_item": <bli_id>,
                               "qty_shipped": 16}
   PATCH shipments/<ship_id>/  {"status": "shipped"}
   ```
6. **Receive** — *UI-only*. On the shipment → **Mark Received** to record receipt against each line and roll the shipment to `received`. Don't PATCH `qty_received` to fake it.
7. **Install** — *UI-only*. On each planned object → **Install → Install from shipment**, which activates the DCIM object and records the shipment that fulfilled it. Don't PATCH DCIM objects to fake the install.

## Brownfield Sparing Workflow

Full detail in [references/workflow-sparing.md](references/workflow-sparing.md). Spares pools hold reserve inventory at a site for replacements:

```bash
POST spares-pools/  {"name": "HQ Cold Storage", "site": <site_id>, "location": <loc_id>}
POST spare-items/   {"pool": <pool_id>,
                     "item_type": {"app_label":"dcim","model":"devicetype"},
                     "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                     "quantity": 5, "status": "serviceable"}
```

- Set `status` to `damaged`/`missing` to flag inventory-audit discrepancies (free-form).
- Serialized single units set `serial`/`asset_tag` with `quantity=1` (quantity >1 with a serial/asset_tag is rejected).
- **Consuming a spare** to replace a failed unit is the install path — **UI-only** (**Install → Install from spares pool** on the planned object). The UI decrements the pool and records the spare→object linkage. Don't decrement `quantity` or activate the DCIM object over the API to fake it; drive the UI if equipped, else hand off to the user.
- A pulled/failed unit can be logged **back** into a pool as a `damaged` SpareItem over the API. There is no dedicated RMA/swap model.

## Anti-Patterns

1. **Skipping required parents.** A PO requires a `bom` + `vendor`; a Shipment requires a `purchase_order` + `courier` + `tracking_number`; line items require their parent + a `bom_line_item`. Create parents first.
2. **Illegal status jumps.** `draft→ordered` etc. return 400 by default. Use `status-choices/` to discover valid targets, or POST a `status-transition-rules/` record before the jump.
3. **Non-draft PO without `order_id`.** Setting a PO to any non-draft status without an `order_id` returns 400.
4. **Mismatched accounts/locations.** `vendor_account.vendor` must equal the PO `vendor`; `courier_account.courier` must equal the Shipment `courier`; a Shipment `location` must belong to its `site`. Mismatch → 400.
5. **Shipment date order.** `date_expected`/`date_received` earlier than `date_shipped` → 400.
6. **Faking generate/receive/install over REST.** They are UI-only — drive the UI (browser/computer-use) or hand off to the user. Never hand-author line items, PATCH `qty_received`, or PATCH DCIM objects to substitute for them (see [UI-Only Operations](#ui-only-operations--do-not-fake-them-over-the-api)).
7. **Hand-authoring BOM line items.** Line items are produced by UI **Generate** from scope rules — author scope rules, not lines. Generated lines are immutable over the API and only change while the BOM is `draft`.
8. **Uniqueness traps.** Unique: Vendor `name`; `(vendor, account_number)`; `(vendor_account, order_id)`; `(courier, tracking_number)`; `(purchase_order, bom_line_item)`; SpareItem `asset_tag` (global); BOM `name`; SparesPool `name`.
9. **Wrong content type for a generic relation.** Line items and spares point at **type** models (`dcim.devicetype`/`moduletype`/`racktype` or the feature's `cabletype`); BOM objects point at **instance** models (`dcim.device`/`rack`/`module`/`cable`).
10. **Assuming receipt activates inventory.** It doesn't — install is a separate UI-only step that activates the planned object and links it to its shipment/spare.

## API Endpoints Summary

| Endpoint (under `/api/plugins/asset-lifecycle/`) | Purpose |
|----------|---------|
| `boms/`, `bom-scope-rules/`, `bom-line-items/`, `bom-objects/` | Bill of materials + scope/lines/objects |
| `cable-types/` | Plugin-local cable type (type+profile); usually auto-created |
| `vendors/`, `vendor-accounts/` | Suppliers + accounts |
| `purchase-orders/`, `po-line-items/` | Orders + lines |
| `couriers/`, `courier-accounts/` | Shippers (UPS/FedEx/DHL seeded) + accounts |
| `shipments/`, `shipment-line-items/` | Deliveries + lines (qty_shipped/qty_received) |
| `spares-pools/`, `spare-items/` | Reserve inventory for replacements |
| `status-transition-rules/` (+ `status-choices/`) | Editable FSM for BOM/PO/Shipment |

## References

- [references/data-model.md](references/data-model.md) — Full object graph, per-model fields, constraints. Load when building requests or decoding validation errors.
- [references/status-transitions.md](references/status-transitions.md) — Default transition rule sets + the BOM "not current" lock. Load on a status 400.
- [references/workflow-greenfield.md](references/workflow-greenfield.md) — Complete procurement walkthrough with payloads; API-driven steps plus the UI-only Generate/Receive/Install handoffs.
- [references/workflow-sparing.md](references/workflow-sparing.md) — Spares pool stocking, audits, and consuming a spare (UI-only install) for a replacement.
