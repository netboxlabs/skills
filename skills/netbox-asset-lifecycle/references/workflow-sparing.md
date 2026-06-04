# Brownfield Sparing — Full Walkthrough

Spares pools hold reserve inventory at a site so failed equipment can be replaced quickly. All paths are under `/api/plugins/asset-lifecycle/`.

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
- `quantity > 1` is rejected when a `serial` or `asset_tag` is set (those identify a single unit).
- `asset_tag` is globally unique; `(item_type, item_id, serial)` is unique when serial is set.
- `item_type` must be a **type** model (`dcim.racktype`/`devicetype`/`moduletype` or `netbox_asset_lifecycle.cabletype`).

## 3. Inventory audits

`SpareItem.status` is free-form (not transition-rule governed). Use it to flag discrepancies during a physical audit:

```bash
PATCH spare-items/<id>/  {"status": "damaged"}   # or "missing"
```

## 4. Consume a spare to replace a failed unit — UI-only

The "spare in" operation is the install path, driven through the UI: on the planned object → **Install → Install from spares pool**. The UI filters available spares to those whose `(item_type, item_id, variant)` exactly match the target and whose `status = serviceable`, then on submit:
- activates the replacement DCIM object,
- links the chosen spare to it (`BOMObject.spare_item`),
- **decrements** the spare's `quantity` by 1, and
- **deletes** the spare item when quantity reaches 0.

This has no REST endpoint. Drive it yourself if you have browser automation or computer-use, otherwise hand off to the user (tell them which object to install and which pool to pull from).

Do **not** decrement `spare-items/<id>/ {"quantity": <n-1>}` and PATCH the DCIM object to `active` over the API to simulate it — that skips the spare→object traceability and risks the pool accounting drifting from reality.

## 5. Returning a failed unit to stock

There is no dedicated RMA/swap model. A pulled or failed unit can be logged **back** into a pool as a `damaged` SpareItem (then handled out-of-band, e.g. RMA to the vendor), while the failed in-service device is decommissioned in NetBox core separately:

```bash
POST spare-items/  {"pool": <pool_id>,
                    "item_type": {"app_label":"dcim","model":"devicetype"},
                    "item_id": <devicetype_id>, "variant": {"airflow":"front-to-rear"},
                    "quantity": 1, "serial": "FOC1234X5YZ", "status": "damaged"}
```
