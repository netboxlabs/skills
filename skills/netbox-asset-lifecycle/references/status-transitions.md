# Asset Lifecycle — Status Transitions Reference (plugin v0.3.x)

Load this when a status change or create returns **400**.

## How the transition system works

`BOM`, `PurchaseOrder`, and `Shipment` each carry a `status`. Changes are validated against **StatusTransitionRule** records for that object type:

- **On create**, the status must match an *initial-status rule* — a rule with a **blank `from_status`** and `to_status` = the requested status. Otherwise: `400 {"status": ["<status> is not a valid initial status."]}`.
- **On update**, a change from `current` to `new` is allowed when either a rule has `from_status = current, to_status = new`, or a rule has `from_status = new, to_status = current` **and** `allow_reverse = true`. Otherwise: `400 {"status": ["Cannot transition from <current> to <new>"]}`.

Rules are ordinary records — list, create, edit, delete via `status-transition-rules/`. `SpareItem.status` is **not** rule-governed.

Discover valid status strings for an object type:

```bash
GET /api/plugins/asset-lifecycle/status-transition-rules/status-choices/?object_type_id=<contenttype_id>
```

(Find the content type id with `GET /api/core/object-types/?app_label=netbox_asset_lifecycle&model=shipment`.)

## Default rules (installed on first migrate)

Format: `from → to (reversible?)`. `(initial)` = blank `from_status`.

### BOM and PurchaseOrder (identical sets)
- `(initial) → draft`
- `draft → approved` (reversible)
- `approved → ordered` (reversible)
- `approved → fulfilled` (reversible)
- `approved → cancelled` (one-way)
- `ordered → fulfilled` (reversible)
- `ordered → cancelled` (one-way)

Reachable by default: `draft ↔ approved ↔ ordered ↔ fulfilled` (and `approved ↔ fulfilled`). Cancellation is only from `approved` or `ordered` and is terminal. There is **no** default `draft → ordered`, `draft → fulfilled`, or `draft → cancelled`, and BOMs/POs can only be **created** as `draft`.

### Shipment
- `(initial) → prepared`
- `(initial) → shipped` *(0.3)*
- `prepared → shipped` (reversible)
- `prepared → cancelled` (one-way)
- `shipped → received` (reversible)
- `shipped → cancelled` (one-way)
- `shipped → lost` (reversible)
- `received → returned` (reversible)
- `lost → received` (reversible)

A shipment can be **created** as `prepared` or `shipped` (not `received`). It can only be cancelled while `prepared` or `shipped`. `received` can revert to `shipped` or advance to `returned`; a `lost` shipment can later become `received`.

## Customising the workflow

```bash
# Permit a direct draft → ordered jump for POs (one-way)
POST status-transition-rules/
{"object_type": {"app_label":"netbox_asset_lifecycle","model":"purchaseorder"},
 "from_status": "draft", "to_status": "ordered", "allow_reverse": false}

# Permit shipments to be created directly as received (initial-status rule)
POST status-transition-rules/
{"object_type": {"app_label":"netbox_asset_lifecycle","model":"shipment"},
 "from_status": "", "to_status": "received"}

# Lock approved BOMs (no going back to draft): clear allow_reverse on draft → approved
PATCH status-transition-rules/<id>/  {"allow_reverse": false}

# Disallow recovering a lost shipment: delete the lost → received rule
DELETE status-transition-rules/<id>/
```

Rules are per object type, so BOM, PO, and shipment lifecycles are tuned independently. Because they are ordinary records they can be exported/imported to keep several NetBox environments consistent.

## Model-specific constraints (also cause 400s)

- **PO `order_id` for `ordered`/`fulfilled`.** Moving a PO to `ordered` or `fulfilled` without an `order_id` is rejected (`approved` does not need one). Send `order_id` in the same PATCH.
- **Received shipment needs a destination.** Moving a shipment to `received` (or creating one as `received` via a custom initial rule) without a `site` and/or `location` is rejected.
- **BOM "not current" lock.** When a generated BOM's scope rules change, `is_current` is cleared and the BOM is **locked in `draft`** — any move out of draft returns 400 until `POST boms/<id>/generate/` runs again. (Ungenerated BOMs are never marked stale.)
- **Scope rules and line items lock on non-draft BOMs.** Creating, editing, or deleting scope rules or BOM line items on an `approved`/`ordered`/`fulfilled`/`cancelled` BOM is rejected. Revert to `draft` first (the default `draft ↔ approved` rule allows it).
- **PO line items lock on non-draft POs.** Same pattern for `po-line-items/`.
- **Archived objects.** Child writes (scope rules, BOM/PO/shipment line items) are rejected while the parent is archived; the UI also blocks status changes on archived objects. `PATCH … {"archived": false}` first.
- **`SpareItem.status`** is free-form (`serviceable`/`damaged`/`missing`) and bypasses the rule system — useful for inventory audits. Only `serviceable` items can be installed or count toward allocations.
