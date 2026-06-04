# Asset Lifecycle — Status Transitions Reference

Load this when a status change returns **400**.

## How the transition system works

`BOM`, `PurchaseOrder`, and `Shipment` each carry a `status`. A status change is validated against **StatusTransitionRule** records for that object type. The change is allowed when either:
- a rule exists with `from_status = current` and `to_status = new`, OR
- a rule exists with `from_status = new` and `to_status = current` **and** `allow_reverse = true`.

If no matching rule exists, the API returns **400** (`Cannot transition from X to Y`). Rules are ordinary records — list, create, edit, or delete them via `status-transition-rules/`. `SpareItem.status` is **not** rule-governed (set it freely).

Discover valid status strings for an object type:

```bash
GET /api/plugins/asset-lifecycle/status-transition-rules/status-choices/?object_type_id=<contenttype_id>
```

## Default rules (shipped with the feature)

Format below: `from → to (reversible?)`.

### BOM and PurchaseOrder (identical sets)
- `draft → approved` (reversible)
- `approved → ordered` (reversible)
- `approved → fulfilled` (reversible)
- `approved → cancelled` (one-way)
- `ordered → fulfilled` (reversible)
- `ordered → cancelled` (one-way)

Reachable by default: `draft ↔ approved ↔ ordered ↔ fulfilled` (and `approved ↔ fulfilled`). Cancellation is only from `approved` or `ordered` and is terminal. There is **no** default `draft → ordered`, `draft → fulfilled`, or `draft → cancelled`.

### Shipment
- `prepared → shipped` (reversible)
- `prepared → cancelled` (one-way)
- `shipped → received` (reversible)
- `shipped → cancelled` (one-way)
- `shipped → lost` (reversible)
- `received → returned` (reversible)
- `lost → received` (reversible)

A shipment can only be **cancelled** while `prepared` or `shipped`. `received` can revert to `shipped` or advance to `returned`; a `lost` shipment can later become `received`.

## Enabling an otherwise-illegal jump

Add a rule, then make the change:

```bash
POST /api/plugins/asset-lifecycle/status-transition-rules/
{"object_type": {"app_label":"netbox_asset_lifecycle","model":"purchaseorder"},
 "from_status": "draft", "to_status": "ordered", "allow_reverse": false}

PATCH /api/plugins/asset-lifecycle/purchase-orders/<id>/  {"status": "ordered"}
```

## Model-specific constraints (also cause 400s)

- **PO needs an order_id when non-draft.** Setting a PurchaseOrder to any status other than `draft` without an `order_id` is rejected. Set `order_id` in the same PATCH.
- **BOM "not current" lock.** When a BOM's scope rules change after its last generation, the BOM is flagged not-current and is **locked in `draft`** — any move out of draft returns 400 until it is regenerated (UI-only). Editing scope rules (parameters, enabled, action, object types) silently invalidates the BOM until you regenerate it in the UI.
- **SpareItem status** is free-form (`serviceable`/`damaged`/`missing`) and bypasses the rule system entirely — useful for inventory audits.
