# Greenfield Procurement — Full Walkthrough

End-to-end procurement of new equipment, plan to install. All paths are under `/api/plugins/asset-lifecycle/`. Replace `<...>` with real pks.

## Prerequisite: plan the equipment in DCIM

The equipment must already exist as NetBox DCIM objects, typically `status=planned`:
- Planned **Devices** carry a `device_type`; planned **Racks** carry a `rack_type`; planned **Cables** carry a `type`. Untyped planned objects are skipped by UI generation.
- Model these with [netbox-data-modeling](../../netbox-data-modeling/SKILL.md) if they don't exist yet.

## 1. Vendor and account

```bash
POST vendors/         {"name": "Acme Networks", "code": "ACME"}
POST vendor-accounts/ {"vendor": <vendor_id>, "account_number": "ACC-001"}
```

## 2. Build the BOM — create the container + scope rules over the API, then Generate in the UI

A BOM rolls planned DCIM objects up into line items, each one an equipment **type + variant** with a quantity. The agent's job over the API is the BOM container and its **scope rules** — the rules that describe which planned objects the BOM covers. Do **not** hand-author line items; they are produced by generation.

```bash
POST boms/            {"name": "DC1 Spine Refresh", "status": "draft"}
POST bom-scope-rules/ {"bom": <bom_id>,
                       "object_types": [{"app_label":"dcim","model":"device"}],
                       "action": "include",
                       "parameters": {"site_id":[3], "role_id":[5], "status":["planned"]}}
# optional exclude rule
POST bom-scope-rules/ {"bom": <bom_id>, "action": "exclude",
                       "parameters": {"tag":["lab"]}}
```

**Generate is UI-only.** On the draft BOM's detail page → **Generate**. Generation resolves the rules, creates a `BOMObject` per matched DCIM object, rolls them up into `auto_generated` BOM line items, and marks the BOM current. Auto-generated lines are immutable over the API.

- **If you have browser automation or computer-use**, open the BOM and click **Generate**, then resume.
- **Otherwise, hand off**: tell the user to open BOM *DC1 Spine Refresh* and click **Generate**, then continue once they confirm.

After generation, read back the line items (`GET bom-line-items/?bom=<bom_id>`) to get the `bom_line_item` ids you'll reference downstream, then approve over the API (status only changes while `draft`):

```bash
PATCH boms/<bom_id>/  {"status": "approved"}
```

## 3. Purchase order and PO line items

A PO **requires** a `bom` and `vendor`. `order_id` is optional while draft but **required** once non-draft.

```bash
POST purchase-orders/ {"vendor": <vendor_id>, "vendor_account": <acct_id>,
                       "bom": <bom_id>, "status": "draft", "currency": "USD"}
POST po-line-items/   {"purchase_order": <po_id>, "bom_line_item": <bli_id>,
                       "qty_ordered": 16, "unit_price": "1200.00"}
                       # total_price is computed; one PO line per BOM line per PO
PATCH purchase-orders/<po_id>/  {"order_id": "PO-7788", "status": "approved"}
# advance further per the default rules: approved → ordered → fulfilled
```

## 4. Shipment and shipment line items

A Shipment **requires** a `purchase_order`, `courier`, and `tracking_number`. UPS/FedEx/DHL couriers are pre-seeded; otherwise create one with a `tracking_url`.

```bash
POST shipments/           {"purchase_order": <po_id>, "courier": <courier_id>,
                           "tracking_number": "1Z999AA10123456784", "status": "prepared",
                           "site": <site_id>, "location": <loc_id>,
                           "date_shipped": "2026-06-01", "date_expected": "2026-06-05"}
POST shipment-line-items/ {"shipment": <ship_id>, "bom_line_item": <bli_id>,
                           "qty_shipped": 16}
PATCH shipments/<ship_id>/  {"status": "shipped"}
```

Validation: `courier_account` (if set) must belong to `courier`; `location` (if set) must belong to `site`; `date_expected`/`date_received` cannot precede `date_shipped`.

## 5. Receive — UI-only

Receiving is driven through the UI: on the shipment's detail page → **Mark Received**, which records receipt against each line (including any shortfall/damage vs `qty_shipped`) and rolls the shipment to `received`. There is no REST endpoint for it.

- **If equipped**, open the shipment and click **Mark Received**.
- **Otherwise, hand off**: ask the user to open the shipment and **Mark Received**, noting any partial quantities, then continue.

Do **not** PATCH `qty_received` or force the shipment to `received` over the API to substitute for this — it loses receive accounting integrity.

## 6. Install — UI-only

Receiving does **not** activate equipment; the DCIM objects still exist as `planned`. Install is a separate UI step: on each planned object → **Install → Install from shipment**, which activates the DCIM object **and** records which shipment fulfilled it (`BOMObject.shipment`). Both the activation and the traceability linkage happen here.

- **If equipped**, open the planned object and run **Install → Install from shipment** (it lists received shipments for the same BOM).
- **Otherwise, hand off**: tell the user which object to install and from which shipment, then continue.

Do **not** PATCH `/api/dcim/devices/<id>/ {"status":"active"}` to fake an install — it skips the shipment→object traceability the feature exists to provide.

## Cabling note

Cables are first-class. A scope rule can target `dcim.cable`; UI generation creates a plugin-local **CableType** (`type` + `profile`) per distinct cable and rolls quantities by variant (`color`, `length`, `length_unit`). Cable BOM line items therefore use `netbox_asset_lifecycle.cabletype` as their `item_type`. Untyped cables are skipped.
