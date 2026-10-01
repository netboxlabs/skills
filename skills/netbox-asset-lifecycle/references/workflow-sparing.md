# Brownfield Sparing — Full Walkthrough (plugin v0.3.x)

Spares pools hold reserve inventory at a site so failed or newly planned equipment can be fulfilled without a purchase. All paths are under `/api/plugins/asset-lifecycle/`.

## 1. Create a spares pool

A pool is named, lives at a **required** site, and optionally at a location:

```bash
POST spares-pools/  {"name": "HQ Cold Storage", "site": <site_id>, "location": <loc_id>}
```

## 2. Stock spare items

Each spare item is one or more units of an equipment **type + variant** in a pool:

```bash
# Bulk, non-serialized
POST spare-items/  {"pool": <pool_id>,
                    "item_type": {"app_label":"dcim","model":"devicetype"},
                    "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                    "quantity": 5, "status": "serviceable"}

# Individually tracked unit
POST spare-items/  {"pool": <pool_id>,
                    "item_type": {"app_label":"dcim","model":"devicetype"},
                    "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                    "quantity": 1, "serial": "FOC9999Z9ZZ", "asset_tag": "SPARE-0001",
                    "status": "serviceable"}
```

Constraints:
- `quantity > 1` is rejected when a `serial` or `asset_tag` is set (those identify a single unit). `quantity` may be 0 (a drained spare).
- `asset_tag` is globally unique; `(item_type, item_id, serial)` is unique when serial is set.
- Non-serialized items are unique on `(pool, item_type, item_id, variant, status)` — to add stock, PATCH `quantity` on the existing row.
- `item_type` must be a **type** model (`dcim.racktype`/`devicetype`/`moduletype` or `netbox_asset_lifecycle.cabletype`).
- `variant` must be a JSON object (`{}` for none). Match the attributes the planned objects will carry — a `{}` spare will not fulfil a planned device whose `airflow` is set.

## 3. Allocations: target stock levels *(0.3)*

An allocation declares the min and/or max quantity of one item + variant a pool should hold. It is **advisory** — nothing is blocked — but the API computes stock state so you don't have to:

```bash
POST spare-item-allocations/  {"pool": <pool_id>,
                               "item_type": {"app_label":"dcim","model":"devicetype"},
                               "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                               "min_quantity": 2, "max_quantity": 8}
# at least one of min/max; max >= min; one allocation per pool + item + variant

GET spare-item-allocations/?pool_id=<pool_id>
# each row: current_quantity (serviceable units of that item+variant in the pool),
#           below_minimum, above_maximum, is_fulfilled  (all read-only)
```

Reorder / rebalance loop for an agent:

```python
import requests
s = requests.Session(); s.headers["Authorization"] = "Token nbt_abc123.xxxxxxxxxxxxxxxx"
base = "https://netbox.example.com/api/plugins/asset-lifecycle/"
r = s.get(base + "spare-item-allocations/", params={"limit": 500}); r.raise_for_status()
for a in r.json()["results"]:
    if a["below_minimum"]:
        print(f'LOW  {a["pool"]["name"]}: {a["item"]["display"]} {a["variant"]} '
              f'{a["current_quantity"]}/{a["min_quantity"]}')
    elif a["above_maximum"]:
        print(f'HIGH {a["pool"]["name"]}: {a["item"]["display"]} {a["variant"]} '
              f'{a["current_quantity"]}/{a["max_quantity"]}')
```

Below-minimum items at one site paired with above-maximum items at another are redistribution candidates; below-minimum with no surplus anywhere feeds a new BOM (a manual line item for the type + variant) and PO.

## 4. Inventory audits

`SpareItem.status` is free-form (not transition-rule governed). Use it to flag discrepancies during a physical audit; only `serviceable` items count toward allocations and can be installed:

```bash
PATCH spare-items/<id>/  {"status": "damaged"}   # or "missing"
```

## 5. Fulfil an existing planned object from a spare

The planned object must be covered by an **asset** (i.e. it was matched by a generated BOM). The install action validates the spare, decrements it atomically, links the asset to the **pool**, and stamps `installed`. Requires `change_asset`, `dcim.change_<model>`, and a write-enabled token.

```bash
# find the asset for the planned object
GET  assets/?assigned_object_type=dcim.device&assigned_object_id=<device_id>
# candidate spares must match item_type + item_id + variant exactly and be serviceable with quantity >= 1
GET  spare-items/?item_type=dcim.devicetype&item_id=<devicetype_id>&status=serviceable
POST assets/<asset_id>/install/  {"spare_item": <spare_id>}
# 200 → asset with spares_pool + installed set; spare quantity decremented (kept at 0 when drained)
# 400 "Selected spare item does not match this object." / "Invalid variant." /
#     "Only serviceable spare items can be installed." / "Insufficient quantity." /
#     "This object has already been installed."
# 409 → another install consumed the last unit concurrently; pick another spare
```

Then update the DCIM object as usual (`PATCH /api/dcim/devices/<id>/ {"status": "active", "serial": "..."}`). The asset records the pool, not the consumed spare row, so provenance survives even if that spare item is later deleted.

Do **not** simulate this by PATCHing `assets/<id>/ {"spares_pool": …}` and decrementing `quantity` by hand — you lose the match validation and the atomic decrement.

## 6. Create a brand-new object from a spare

When a spare is being deployed somewhere that was never planned in NetBox (no BOM, no asset), create the object straight from the spare. The plugin seeds the type FK, `serial`, `asset_tag`, and variant attributes (e.g. `airflow`) from the spare; the body supplies the rest of the DCIM object's required fields and may override the seeded ones. Requires `change_spareitem`, `dcim.add_<model>`, and a write-enabled token. Device, rack, and module types only — **cables cannot be installed this way**.

```bash
POST spare-items/<spare_id>/install/
{"name": "hq-leaf-07", "site": <site_id>, "role": <role_id>, "status": "active", "rack": <rack_id>, "position": 12, "face": "front"}
# 201 → the new dcim.Device (standard device serializer); spare quantity decremented
# 400 "Only serviceable spare items can be installed." / "Insufficient quantity." /
#     "Spare items of type … cannot be installed."   (cable types)
# 400 with dcim field errors if the body is missing required device/rack/module fields
```

No asset is created by this path — the object has no BOM/shipment provenance, only the audit trail on the spare.

## 7. Returning a failed unit to stock

There is no dedicated RMA/swap model. A pulled or failed unit can be logged **back** into a pool as a `damaged` SpareItem (then handled out-of-band, e.g. RMA to the vendor), while the failed in-service device is decommissioned in NetBox core separately:

```bash
POST spare-items/  {"pool": <pool_id>,
                    "item_type": {"app_label":"dcim","model":"devicetype"},
                    "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                    "quantity": 1, "serial": "FOC1234X5YZ", "status": "damaged"}
```

Once repaired, `PATCH spare-items/<id>/ {"status": "serviceable"}` returns it to the pool's usable stock (and to the allocation's `current_quantity`).
