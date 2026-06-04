# Asset Lifecycle — Data Model Reference

Every model lives under `/api/plugins/asset-lifecycle/`. Fields below are marked **(W)** writable or **(RO)** read-only. Every model also exposes the standard NetBox fields: `id` (RO), `url` (RO), `display` (RO), `created`/`last_updated` (RO). Models marked *full NetBox model* additionally support `tags` (W), `custom_fields` (W), `comments` (W); models marked *validated-only* do not.

## Object graph

```
dcim.Site ──< SparesPool >── dcim.Location (optional)
SparesPool ──< SpareItem
SpareItem.item → dcim.RackType | DeviceType | ModuleType | CableType   (generic relation)

BOM ──< BOMScopeRule        BOMScopeRule.object_types → ContentType{rack,device,module,cable} (many)
BOM ──< BOMLineItem         BOMLineItem.item → RackType|DeviceType|ModuleType|CableType (generic)
BOM ──< BOMObject           BOMObject.assigned_object → Rack|Device|Module|Cable (generic)
                            BOMObject.shipment / .spare_item = install linkage (UI-only, not serialized)

Vendor ──< VendorAccount
Vendor ──< PurchaseOrder        VendorAccount ──< PurchaseOrder (optional)
BOM ──< PurchaseOrder           # PO REQUIRES a BOM
PurchaseOrder ──< POLineItem    POLineItem.bom_line_item → BOMLineItem

Courier ──< CourierAccount
PurchaseOrder ──< Shipment      # Shipment REQUIRES a PO
Courier ──< Shipment            CourierAccount ──< Shipment (optional)
dcim.Site/Location → Shipment (optional destination)
Shipment ──< ShipmentLineItem   ShipmentLineItem.bom_line_item → BOMLineItem

StatusTransitionRule.object_type → ContentType{bom, purchaseorder, shipment}
```

## BOM models

**BOM** (full NetBox model) — `boms/`
- `name` (W, **unique**), `status` (W, choice), `description` (W), `comments` (W)
- `last_generated` (RO), `is_current` (RO, system-managed — see status-transitions.md for the "not current" lock)

**BOMScopeRule** (full NetBox model) — `bom-scope-rules/`
- `bom` (W, pk), `object_types` (W, **many** ContentTypes: `dcim.rack`/`device`/`module`/`cable`), `enabled` (W, bool, default true), `action` (W, `include`|`exclude`, default `include`), `parameters` (W, **JSON of NetBox list filters**, e.g. `{"site_id":[3],"role_id":[5],"status":["planned"]}`), `description`, `comments`
- Scope rules feed UI generation; over the API they describe intended scope but do not auto-create line items.

**BOMLineItem** (validated-only) — `bom-line-items/`
- `bom` (W, pk), `quantity` (W, int ≥1), `item_type` (W, ContentType — `dcim.racktype`/`devicetype`/`moduletype` or `netbox_asset_lifecycle.cabletype`), `item_id` (W, int), `variant` (W, JSON or null), `item` (RO)
- `auto_generated` (RO) — true for UI-generated rows (immutable); false for manual rows
- Unique: `(bom, item_type, item_id, variant, auto_generated)`. Editable only while the BOM is `draft`.

**BOMObject** (validated-only) — `bom-objects/`
- `bom` (W, pk), `assigned_object_type` (W, ContentType — `dcim.rack`/`device`/`module`/`cable`), `assigned_object_id` (W, int), `assigned_object` (RO)
- The model's `shipment`/`spare_item` install FKs are **not in the serializer** — install linkage cannot be set/read over REST.

**CableType** (full NetBox model) — `cable-types/`
- `type` (W, NetBox cable type choice, e.g. `cat6`, `mmf-om4`), `profile` (W, cable profile choice, may be blank), `description`, `comments`. Unique `(type, profile)`. Normally auto-created during UI generation; rarely created by hand. Cable line items reference `netbox_asset_lifecycle.cabletype` as their `item_type`.

## Procurement models

**Vendor** (full NetBox model) — `vendors/`
- `name` (W, **unique**), `code` (W, optional, unique-if-set), `description`, `comments`. Supports contacts.

**VendorAccount** (full NetBox model) — `vendor-accounts/`
- `vendor` (W, pk), `account_number` (W), `description`, `comments`. Unique `(vendor, account_number)`.

**PurchaseOrder** (full NetBox model) — `purchase-orders/`
- `vendor` (W, pk, required), `vendor_account` (W, pk, optional — **must belong to vendor**), `bom` (W, pk, **required**), `order_id` (W, optional while draft / **required when status != draft**), `status` (W, default `draft`), `currency` (W, ISO code or blank), `description`, `comments`. Unique `(vendor_account, order_id)`.

**POLineItem** (validated-only) — `po-line-items/`
- `purchase_order` (W, pk), `bom_line_item` (W, pk), `qty_ordered` (W, int ≥1, required), `unit_price` (W, decimal ≥0, optional), `total_price` (RO, computed). Unique `(purchase_order, bom_line_item)`.

## Shipping models

**Courier** (full NetBox model) — `couriers/`
- `name` (W, unique), `code` (W, optional unique-if-set), `tracking_url` (W, URL — tracking number is appended to build the link), `description`, `comments`. Supports contacts. **UPS / FedEx / DHL are pre-seeded** with working tracking URLs.

**CourierAccount** (full NetBox model) — `courier-accounts/`
- `courier` (W, pk), `account_number` (W), `description`, `comments`. Unique `(courier, account_number)`.

**Shipment** (full NetBox model) — `shipments/`
- `purchase_order` (W, pk, required), `courier` (W, pk, required), `courier_account` (W, pk, optional — **must belong to courier**), `site` (W, pk, optional dest), `location` (W, pk, optional — **must belong to site**), `tracking_number` (W, required), `status` (W, default `prepared`), `date_shipped`/`date_expected`/`date_received` (W, dates — expected/received cannot precede shipped), `description`, `comments`. Unique `(courier, tracking_number)`.

**ShipmentLineItem** (validated-only) — `shipment-line-items/`
- `shipment` (W, pk), `bom_line_item` (W, pk), `qty_shipped` (W, int ≥1, required), `qty_received` (W, int ≥1, optional). `qty_received` is populated by the UI **Mark Received** flow — don't PATCH it directly to fake receiving (see SKILL.md UI-Only Operations).

## Spares models

**SparesPool** (full NetBox model) — `spares-pools/`
- `name` (W, unique), `site` (W, pk, **required**), `location` (W, pk, optional), `description`, `comments`. Unique `(location, name)`.

**SpareItem** (full NetBox model) — `spare-items/`
- `pool` (W, pk), `item_type` (W, ContentType — racktype/devicetype/moduletype/cabletype), `item_id` (W, int), `item` (RO), `variant` (W, JSON or null), `status` (W, `serviceable`|`damaged`|`missing`, default `serviceable`), `quantity` (W, int ≥1, default 1), `serial` (W, optional), `asset_tag` (W, optional, **globally unique**), `description`, `comments`
- Constraints: quantity >1 is rejected when a serial or asset_tag is set; `(item_type, item_id, serial)` unique when serial is set.

## StatusTransitionRule

**StatusTransitionRule** (full NetBox model) — `status-transition-rules/`
- `object_type` (W, ContentType — only `netbox_asset_lifecycle.bom`/`purchaseorder`/`shipment`), `from_status` (W), `to_status` (W, must differ from `from_status`), `allow_reverse` (W, bool, default true), `description`, `comments`. Unique `(object_type, from_status, to_status)`.
- Custom action: `GET status-transition-rules/status-choices/?object_type_id=<id>` → `[{"value","display"}]` valid statuses for that object type.

## Variant attributes

The `variant` JSON distinguishes line items / spares of the same type. Allowed keys per type:
- Rack: none
- Device: `airflow`
- Module: `airflow`
- Cable: `color`, `length`, `length_unit` (the cable's `type`/`profile` live on the CableType, not the variant)

A spare can fulfill a BOM object only when `item_type` + `item_id` + `variant` match exactly **and** the spare is `serviceable`.
