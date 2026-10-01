# Asset Lifecycle — Data Model Reference (plugin v0.3.x)

Every model lives under `/api/plugins/asset-lifecycle/`. Fields below are marked **(W)** writable or **(RO)** read-only. Every model also exposes the standard NetBox fields: `id`, `url`, `display` (RO). Models marked *full NetBox model* additionally support `tags`, `custom_fields`, `description`, `comments` (W) and `created`/`last_updated` (RO); models marked *validated-only* expose only their own fields.

## Object graph

```
dcim.Site ──< SparesPool >── dcim.Location (optional)
SparesPool ──< SpareItem              SpareItem.item → RackType | DeviceType | ModuleType | CableType (generic)
SparesPool ──< SpareItemAllocation    same item/variant shape; min/max thresholds, advisory

BOM ──< BOMScopeRule        BOMScopeRule.object_types → ContentType{rack,device,module,cable} (many)
BOM ──< BOMLineItem         BOMLineItem.item → RackType|DeviceType|ModuleType|CableType (generic)
BOM ──< Asset               Asset.assigned_object → Rack|Device|Module|Cable (generic)
                            Asset.shipment | Asset.spares_pool = install linkage; Asset.installed = timestamp

Vendor ──< VendorAccount
Vendor ──< PurchaseOrder        VendorAccount ──< PurchaseOrder (optional)
BOM ──< PurchaseOrder           # PO REQUIRES a BOM; BOM cannot be changed after creation
PurchaseOrder ──< POLineItem    POLineItem.bom_line_item → BOMLineItem

Courier ──< CourierAccount
PurchaseOrder ──< Shipment      # Shipment REQUIRES a PO
Courier ──< Shipment            CourierAccount ──< Shipment (optional)
dcim.Site/Location → Shipment (optional destination; required once received)
Shipment ──< ShipmentLineItem   ShipmentLineItem.bom_line_item → BOMLineItem

StatusTransitionRule.object_type → ContentType{bom, purchaseorder, shipment}
```

## BOM models

**BOM** (full NetBox model) — `boms/`
- `name` (W, **unique**), `status` (W, choice; default `draft`), `archived` (W, bool, default false — cannot be true on create)
- `last_generated` (RO), `is_current` (RO — cleared when a scope rule changes; restored by generate; a not-current BOM is locked in `draft`)
- Filters: `name`, `status`, `is_current`, `archived`, `last_generated`, `q`
- Action: `POST boms/<id>/generate/` — see SKILL.md.

**BOMScopeRule** (full NetBox model) — `bom-scope-rules/`
- `bom` (W, pk), `object_types` (W, **many** ContentTypes: `dcim.rack`/`device`/`module`/`cable`), `enabled` (W, bool, default true), `action` (W, `include`|`exclude`, default `include`), `parameters` (W, **JSON of the target model's list filters**, e.g. `{"site_id":[3],"role_id":[5],"status":["planned"]}`; `{}` matches everything of the selected types)
- Writable (create/edit/delete) **only while the BOM is `draft` and not archived**. Any change marks a generated BOM not-current.
- Evaluation: per object type, union of `include` rules, minus union of `exclude` rules. Untyped cables and racks are dropped.

**BOMLineItem** (validated-only) — `bom-line-items/`
- `bom` (W, pk), `quantity` (W, int ≥1, default 1), `item_type` (W, ContentType — `dcim.racktype`/`devicetype`/`moduletype` or `netbox_asset_lifecycle.cabletype`), `item_id` (W, int), `variant` (W, JSON object, never null — omit or `{}`), `item` (RO)
- `auto_generated` (RO) — true for generated rows: **immutable**, deleted and recreated on every generate. Manual rows (false) survive regeneration.
- Unique `(bom, item_type, item_id, variant, auto_generated)`. Create/edit/delete only while the BOM is `draft` and not archived.
- Filters: `bom_id`, `item_type`, `item_id`, `auto_generated`, `quantity`.

**Asset** (validated-only) — `assets/` *(was `bom-objects/` / `BOMObject` before 0.3)*
- `bom` (W, pk), `assigned_object_type` (W, ContentType — `dcim.rack`/`device`/`module`/`cable`), `assigned_object_id` (W, int), `assigned_object` (RO)
- `shipment` (W, pk or null), `spares_pool` (W, pk or null) — the fulfilment linkage. **Set these via `POST assets/<id>/install/`**, not by PATCH: the action validates the shipment/spare and consumes inventory. Only one of the two is set for an installed asset.
- `installed` (RO datetime) — stamped when a shipment or pool is first linked; cleared if the link is removed (including when the shipment or pool is deleted).
- Assets are rebuilt from scratch on every BOM generate (existing links are lost — install only after the BOM is approved).
- Filters: `bom_id`, `shipment_id`, `spares_pool_id`, `assigned_object_type`, `assigned_object_id`, `installed_after`, `installed_before`, `installed__isnull`, `q` (BOM name).

**CableType** (full NetBox model) — `cable-types/`
- `type` (W, NetBox cable type choice, e.g. `cat6a`, `mmf-om4`), `profile` (W, NetBox cable profile choice, may be blank), `connector_a` / `connector_b` (W *(0.3)*, connector choices, may be blank — power cables accept NetBox power port/outlet connector values; all other media accept interface port connector values; a mismatch is rejected)
- Unique `(type, profile, connector_a, connector_b)`.
- Auto-created during generation with **blank connectors** (NetBox cables carry no connector data) — fill connectors in afterward, or create fully-specified types by hand for ordering. Cable line items and spares reference `netbox_asset_lifecycle.cabletype` as `item_type`.

## Procurement models

**Vendor** (full NetBox model) — `vendors/`
- `name` (W, **unique**), `code` (W, optional, unique when set). Supports contacts.

**VendorAccount** (full NetBox model) — `vendor-accounts/`
- `vendor` (W, pk), `account_number` (W). Unique `(vendor, account_number)`.

**PurchaseOrder** (full NetBox model) — `purchase-orders/`
- `vendor` (W, pk, required), `vendor_account` (W, pk, optional — **must belong to vendor**), `bom` (W, pk, **required, immutable after create**), `order_id` (W, optional; **required for `ordered`/`fulfilled`**), `status` (W, default `draft`), `currency` (W, ISO-4217 choice from a fixed list — USD, EUR, GBP, JPY, CAD, AUD, CHF, CNY, HKD, SGD, INR, MXN, BRL, KRW, SEK, NOK, DKK, NZD, ZAR, AED — or blank; defaults to the `default_currency` plugin setting when omitted), `archived` (W, bool)
- Unique `(vendor, order_id)` when `order_id` is set.
- Action: `GET purchase-orders/<id>/export-line-items/?format=json|csv|yaml` — header (order_id, vendor, vendor_account, currency, status, bom) + `line_items[]` (item_type, manufacturer, model, part_number, variant, qty_ordered, unit_price, total_price).
- Filters: `vendor_id`, `vendor_account_id`, `bom_id`, `bom_line_item_id`, `currency`, `order_id`, `status`, `archived`, `q`.

**POLineItem** (validated-only) — `po-line-items/`
- `purchase_order` (W, pk), `bom_line_item` (W, pk), `qty_ordered` (W, int ≥1, required), `unit_price` (W, decimal ≥0, optional), `total_price` (RO, computed). Unique `(purchase_order, bom_line_item)`.
- Create/edit/delete **only while the PO is `draft`** and not archived.

## Shipping models

**Courier** (full NetBox model) — `couriers/`
- `name` (W, unique), `code` (W, optional, unique when set), `tracking_url` (W, URL prefix — tracking number is appended). Supports contacts. **UPS / FedEx / DHL are seeded** with working tracking URLs (codes `ups`, `fedex`, `dhl`).

**CourierAccount** (full NetBox model) — `courier-accounts/`
- `courier` (W, pk), `account_number` (W). Unique `(courier, account_number)`.

**Shipment** (full NetBox model) — `shipments/`
- `purchase_order` (W, pk, required), `courier` (W, pk, required), `courier_account` (W, pk, optional — **must belong to courier**), `site` (W, pk, optional), `location` (W, pk, optional — **must belong to site**), `tracking_number` (W, required), `status` (W; **create as `prepared` or `shipped`**), `date_shipped`/`date_expected`/`date_received` (W, dates — expected/received cannot precede shipped), `archived` (W, bool)
- **`received` requires `site` and/or `location`.**
- Unique `(courier, tracking_number)`.
- Filters: `bom_id` (via the PO), `purchase_order_id`, `bom_line_item_id`, `courier_id`, `courier_account_id`, `site_id`, `location_id`, `tracking_number`, `status`, `date_*`, `archived`, `q`.

**ShipmentLineItem** (validated-only) — `shipment-line-items/`
- `shipment` (W on create, then locked), `bom_line_item` (W on create, then locked), `qty_shipped` (W, int ≥1, required), `qty_received` (W, int ≥1, optional — set it on receipt; lower than `qty_shipped` records a shortfall). To change shipment or line, delete and recreate.
- Locked while the shipment is archived. A PO line's quantity may be split across several shipments.

## Spares models

**SparesPool** (full NetBox model) — `spares-pools/`
- `name` (W, unique), `site` (W, pk, **required**), `location` (W, pk, optional). Unique `(location, name)`.
- Deleting a pool nulls `Asset.spares_pool` on assets installed from it (their `installed` stamp clears unless a shipment is also linked). Deleting an individual spare item has no effect on assets.

**SpareItem** (full NetBox model) — `spare-items/`
- `pool` (W, pk), `item_type` (W, ContentType — racktype/devicetype/moduletype/cabletype), `item_id` (W, int), `item` (RO), `variant` (W, JSON object, never null), `status` (W, `serviceable`|`damaged`|`missing`, default `serviceable`; not rule-governed), `quantity` (W, int **≥0**, default 1 — install drains to 0 and keeps the record), `serial` (W, optional), `asset_tag` (W, optional, **globally unique**)
- Constraints: quantity >1 rejected when `serial` or `asset_tag` is set; `(item_type, item_id, serial)` unique when serial set; non-serialized items unique on `(pool, item_type, item_id, variant, status)` — bump `quantity` on the existing row rather than adding a duplicate.
- Action: `POST spare-items/<id>/install/` — creates a new Device/Rack/Module from the spare (see workflow-sparing.md).
- Filters: `pool_id`, `item_type`, `item_id`, `status`, `serial`, `asset_tag`, `quantity`, `q`.

**SpareItemAllocation** (full NetBox model) — `spare-item-allocations/` *(0.3)*
- `pool` (W, pk), `item_type`/`item_id`/`variant` (W, same shape as SpareItem), `item` (RO), `min_quantity` (W, int or null, default 1), `max_quantity` (W, int or null)
- RO computed: `current_quantity` (sum of **serviceable** spare items in the pool with the same item + variant), `below_minimum`, `above_maximum`, `is_fulfilled`
- Constraints: at least one of min/max; `max_quantity ≥ min_quantity`; unique `(pool, item_type, item_id, variant)`.
- **Advisory** — thresholds do not block adding or consuming spares; they surface low-stock / excess-stock conditions (UI "Threshold Violations" panel on the pool; API flags).
- Filters: `pool_id`, `item_type`, `item_id`, `min_quantity`, `max_quantity`.

## StatusTransitionRule

**StatusTransitionRule** (full NetBox model) — `status-transition-rules/`
- `object_type` (W, ContentType — only `netbox_asset_lifecycle.bom`/`purchaseorder`/`shipment`), `from_status` (W; **blank = initial-status rule**), `to_status` (W, must differ from `from_status`; must be a valid status for the object type), `allow_reverse` (W, bool, default true; forced false on initial-status rules)
- Unique `(object_type, from_status, to_status)`.
- Custom action: `GET status-transition-rules/status-choices/?object_type_id=<id>` → `[{"value","display"}]`.

## Variant attributes

The `variant` JSON distinguishes line items / spares / allocations of the same type. Allowed keys per type:
- Rack: none
- Device: `airflow`
- Module: `airflow`
- Cable: `color`, `length`, `length_unit` (the cable's `type`/`profile`/connectors live on the CableType, not the variant)

Blank/null attributes are dropped on save; an empty variant is stored as `{}`. Variants are compared for **exact equality** — a spare fulfils an asset only when `item_type` + `item_id` + normalized `variant` match and the spare is `serviceable` with `quantity ≥ 1`. Generation derives the variant from the planned object's own fields (e.g. `Device.airflow`), so stock spares with the same attributes the planned objects carry.
