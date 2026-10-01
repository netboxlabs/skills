# Greenfield Procurement — Full Walkthrough (plugin v0.3.x)

End-to-end procurement of new equipment, plan to install, entirely over the REST API. All paths are under `/api/plugins/asset-lifecycle/`. Replace `<...>` with real pks.

## Prerequisite: plan the equipment in DCIM

The equipment must already exist as NetBox DCIM objects, typically `status=planned`:
- Planned **Devices** carry a `device_type` (and, if you will spare against them, an `airflow`); planned **Racks** carry a `rack_type`; planned **Cables** carry a `type`. Untyped racks and cables are silently skipped by generation.
- Model these with [netbox-data-modeling](../../netbox-data-modeling/SKILL.md) if they don't exist yet.

## 1. Vendor and account

```bash
POST vendors/         {"name": "Acme Networks", "code": "ACME"}
POST vendor-accounts/ {"vendor": <vendor_id>, "account_number": "ACC-001"}
```

## 2. Build the BOM: container + scope rules, then generate

A BOM rolls planned DCIM objects up into line items, each one an equipment **type + variant** with a quantity, and records one **asset** per matched planned object. Author the container and its **scope rules**; do **not** hand-author line items.

```bash
POST boms/            {"name": "DC1 Spine Refresh"}          # status defaults to draft (the only initial status)
POST bom-scope-rules/ {"bom": <bom_id>,
                       "object_types": [{"app_label":"dcim","model":"device"}],
                       "action": "include",
                       "parameters": {"site_id":[3], "role_id":[5], "status":["planned"]}}
# optional exclude rule (applied after includes for the same object type)
POST bom-scope-rules/ {"bom": <bom_id>, "action": "exclude",
                       "object_types": [{"app_label":"dcim","model":"device"}],
                       "parameters": {"tag":["lab"]}}
```

`parameters` accepts whatever the corresponding NetBox list endpoint accepts (`/api/dcim/devices/?site_id=3&role_id=5&status=planned`), so test a rule's filter against the core API first.

Generate:

```bash
POST boms/<bom_id>/generate/          # empty body; needs change_bom + a write-enabled token
# 200 → the BOM with last_generated set and is_current=true
# 400 "BOM must be in draft status to generate." / "BOM has no enabled scope rules. …"
```

Generation deletes all assets and auto-generated line items, re-evaluates enabled rules, creates one `Asset` per matched object, and rolls them up into `auto_generated` line items grouped by type + variant. Manual line items survive. It is idempotent — regenerate as often as the design changes while the BOM is `draft`.

Read back and approve:

```bash
GET  bom-line-items/?bom_id=<bom_id>      # note each line's id, item, variant, quantity
GET  assets/?bom_id=<bom_id>              # one per planned object
PATCH boms/<bom_id>/  {"status": "approved"}
```

Approval locks scope rules and line items. Editing a scope rule after generation marks the BOM not-current and pins it to `draft` until regenerated. A one-off item not modeled in NetBox can be added as a manual line (`POST bom-line-items/ {"bom": …, "item_type": …, "item_id": …, "quantity": 2}`) while the BOM is draft.

## 3. Purchase order and PO line items

A PO **requires** a `bom` (immutable afterward) and `vendor`. `order_id` is optional until the PO moves to `ordered`. Line items are editable **only while the PO is `draft`**. If `currency` is omitted, the `default_currency` plugin setting applies.

```bash
POST purchase-orders/ {"vendor": <vendor_id>, "vendor_account": <acct_id>,
                       "bom": <bom_id>, "currency": "USD"}                # status defaults to draft
POST po-line-items/   {"purchase_order": <po_id>, "bom_line_item": <bli_id>,
                       "qty_ordered": 16, "unit_price": "1200.00"}
                       # total_price is computed; one PO line per BOM line per PO
PATCH purchase-orders/<po_id>/  {"status": "approved"}                    # locks the lines
GET   purchase-orders/<po_id>/export-line-items/?format=csv               # hand to the vendor
PATCH purchase-orders/<po_id>/  {"order_id": "PO-7788", "status": "ordered"}
```

Split a BOM across vendors by creating several POs against the same BOM, each taking a subset of line items (a BOM line can appear on many POs, once per PO).

## 4. Shipment and shipment line items

A Shipment **requires** a `purchase_order`, `courier`, and `tracking_number`. Create it as `prepared` (staged at the vendor) or `shipped` (already in transit). UPS/FedEx/DHL couriers are seeded; otherwise create one with a `tracking_url`. Create shipments once the PO is `ordered` — that is the documented workflow and the UI only offers *Create Shipment* on ordered POs.

```bash
POST shipments/           {"purchase_order": <po_id>, "courier": <courier_id>,
                           "tracking_number": "1Z999AA10123456784", "status": "shipped",
                           "site": <site_id>, "location": <loc_id>,
                           "date_shipped": "2026-09-01", "date_expected": "2026-09-05"}
POST shipment-line-items/ {"shipment": <ship_id>, "bom_line_item": <bli_id>, "qty_shipped": 16}
```

Validation: `courier_account` (if set) must belong to `courier`; `location` (if set) must belong to `site`; `date_expected`/`date_received` cannot precede `date_shipped`; `(courier, tracking_number)` is unique. A PO line with quantity >1 can be split across several shipments. `shipment` and `bom_line_item` on a line item are locked after create.

## 5. Receive

Receiving is ordinary field editing; the UI's **Mark Received** button is a bulk form for the same writes.

```bash
PATCH shipments/<ship_id>/  {"status": "received", "date_received": "2026-09-05"}
        # 400 unless the shipment has a site and/or location
PATCH shipment-line-items/<sli_id>/  {"qty_received": 15}     # < qty_shipped records a shortfall/damage
```

Receiving does **not** activate equipment — the DCIM objects still exist as `planned`.

## 6. Install

Install links each asset to the received shipment (traceability) and stamps `installed`. It requires `change_asset`, `dcim.change_<model>` on the target object, and a write-enabled token. The shipment must be `received` and belong to a PO on the asset's BOM.

```bash
GET  assets/?bom_id=<bom_id>&installed__isnull=true          # what is still un-installed
POST assets/<asset_id>/install/  {"shipment": <ship_id>}
# 200 → asset with shipment + installed set
# 400 "Shipment has not been received." / "Shipment does not belong to this object's BOM." /
#     "This object has already been installed."
```

The REST install does **not** modify the DCIM object. Update it yourself as the physical work happens:

```bash
PATCH /api/dcim/devices/<device_id>/  {"status": "staged", "serial": "FOC2345X6YZ", "asset_tag": "NB-0042"}
```

(The UI install form does both in one submit and lets you set status/serial/asset tag inline.) The installed object's detail page shows a **Procurement** panel with the BOM, shipment, and PO.

## 7. Close out and archive

```bash
PATCH purchase-orders/<po_id>/  {"status": "fulfilled"}
PATCH boms/<bom_id>/            {"status": "fulfilled"}
PATCH shipments/<ship_id>/      {"archived": true}
PATCH purchase-orders/<po_id>/  {"archived": true}
PATCH boms/<bom_id>/            {"archived": true}
```

Archived objects keep their status and data, disappear from UI lists (REST still returns them unless `?archived=false`), and reject child writes until unarchived.

## Cabling note

Cables are first-class. A scope rule can target `dcim.cable`; generation creates (or reuses) a plugin-local **CableType** per distinct `type` + `profile` — always with **blank connectors**, since NetBox cables carry none — and rolls quantities by variant (`color`, `length`, `length_unit`). Cable line items therefore use `netbox_asset_lifecycle.cabletype` as `item_type`. To order a specific SKU, either fill `connector_a`/`connector_b` on the auto-created type afterward, or create a fully-specified cable type and a manual line item for it.
