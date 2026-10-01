---
name: netbox-ndx
description: >
  NetBox Data Exchange (NDX) — a curated catalog of infrastructure component metadata.
  The device/module/rack type definitions are open to the world (browse + manual YAML
  download); the "enrichment data" (lifecycle dates, thermal, environmental, operational,
  platform/NOS, observability protocol metadata) and the built-in NDX features in NetBox
  Cloud/Enterprise are commercial. Use when consuming enrichment data via the API, running
  EOL/lifecycle analyses, importing device types, or understanding how NDX data is structured.
license: Apache-2.0
---

# NetBox Data Exchange (NDX)

**NDX (NetBox Data Exchange)** is a curated, continuously-updated catalog of infrastructure **component metadata** — covering tens of thousands of device types across hundreds of manufacturers. It replaces hours of digging through vendor PDFs and EOL bulletins with a searchable reference dataset.

> **Open catalog, commercial enrichment + in-product features.** The **catalog of device/module/rack type definitions is open to the world** — anyone, including community users, can browse the public catalog and download YAML type definitions manually (same shape as the community Device Type Library). What's **commercial** is (a) the **enrichment data** — lifecycle/thermal/operational/platform/observability metadata — and (b) the **built-in NDX features inside NetBox Cloud / NetBox Enterprise** that browse, import, sync, and attach enrichment from within the product. Keep this boundary clear when you describe NDX.

> **NDX is a feature, not a plugin.** Refer to the in-product capability as "the NDX feature." Its plugin implementation is an internal detail — focus on what it delivers and how to consume it via the API.

> **Your knowledge of NDX may be outdated.** Catalog coverage, enrichment fields, and tier behavior evolve continuously. The data is AI-assisted-curated and carries provenance/confidence — prefer retrieval over pre-trained knowledge and always surface provenance when presenting values.

## What NDX Delivers — Two Data Layers

| Layer | What it is | Access | Where it goes in NetBox |
|-------|-----------|--------|------------------------|
| **Device-type definitions** | Physical topology of a device/module/rack type: interfaces, power ports, console ports, bays, rack units, weight. Same shape as the community Device Type Library (DTL), included in full with attribution, plus thousands of NetBox Labs-curated definitions. | **Open** — browse + manual YAML download by anyone, including community users | Standard NetBox `DeviceType`/`ModuleType`/`RackType` objects with component templates |
| **Enrichment data** | New metadata the DTL never carried: **lifecycle dates, thermal, environmental limits, operational power/noise, platform/NOS, and observability protocol metadata** (SNMP/gNMI/NETCONF/Redfish/Modbus/etc.), each value carrying provenance + a confidence score. The commercial value-add. | **Commercial** — values delivered through the in-product NDX feature on paid plans | The enrichment record attached to each imported type |

### What NDX is NOT
- **Not monitoring / not telemetry.** No real-time state.
- **Not a CMDB / not instance data.** NDX describes device **types** ("a Cisco Nexus 93180YC-FX3"), never your specific running unit.
- **Not auto-configuration.** Observability metadata tells you *what* and *how* to monitor (OIDs, paths); it does not configure tools.
- **Not guaranteed complete.** Coverage varies; many fields are null. Provenance and confidence expose this.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| NDX feature docs | `https://netboxlabs.com/docs/ndx/` | Feature reference, import/sync, tiers |
| NDX public catalog | `https://netboxlabs.com/ndx` | Browse/search the catalog; YAML download (enrichment values gated) |
| NetBox DCIM models | `https://netboxlabs.com/docs/netbox/models/dcim/` | DeviceType/ModuleType/RackType structure |
| Community DTL | `https://github.com/netbox-community/devicetype-library` | The open-source DTL NDX includes (Apache-2.0) |
| NetBox MCP / Platform MCP | If configured — list/inspect `/api/plugins/ndx/` endpoints | Verify live endpoints and enrichment fields |

## FIRST: Verify the Feature Is Available

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" \
  "$NETBOX_URL/api/plugins/ndx/import-records/?limit=1" | python -m json.tool
```

A paginated response means the in-product NDX feature is available. **404** = the feature is not enabled on this instance (the catalog is still browsable + downloadable as YAML at the public catalog, just not wired into this NetBox). The in-product feature ships with NetBox Cloud (all plans) and NetBox Enterprise; free plans see the catalog, import, and enrichment **availability flags**, while paid plans see full enrichment **values** (see Tiers below).

---

## Scope

This skill covers **consuming NDX data via the NetBox REST API** (`/api/plugins/ndx/`) — the surface any agent with a NetBox token can reach — and understanding the enrichment data model. It does NOT cover:
- Designing your own data model → use [netbox-data-modeling](../netbox-data-modeling/SKILL.md)
- Bulk-migrating inventory → use [netbox-migration](../netbox-migration/SKILL.md)
- Core REST patterns/auth → use [netbox-api-integration](../netbox-api-integration/SKILL.md)

**Posture: NDX is read/consume.** The authoritative catalog is curated centrally by NetBox Labs; your instance pulls it in. Agents **look up and consume** enrichment + lifecycle data; **import** is a secondary action. You do not author enrichment values.

## REST API Surface

Base prefix **`/api/plugins/ndx/`**. Standard NetBox token auth and DRF pagination (`count`/`next`/`previous`/`results`, `?limit=`/`?offset=`).

| Endpoint | Methods | Purpose |
|----------|---------|---------|
| `import-records/` | GET (, PATCH) | The primary lookup table — maps each imported type to its NetBox object and carries enrichment inline |
| `import-records/{id}/` | GET, PATCH | One import record |
| `enrichments/` | GET (, PATCH) | Enrichment records directly (filter by `has_*` flags) |
| `enrichments/{id}/` | GET, PATCH | One enrichment record |
| `import/` | POST | Import one or more catalog entries into NetBox |
| `sync/` | POST | Queue a background update-check (flags `update_available`; does not apply) |

> Treat `import-records` and `enrichments` as **read** endpoints — records are populated by the import pipeline, not hand-authored.

### `GET import-records/` — the entry point

Each record links an NDX catalog identity to the local NetBox type and nests the full enrichment:
- `ndx_id` — stable identifier, `vendor_slug/slug` (e.g. `cisco/nexus-93180yc-fx3`)
- `ndx_manufacturer`, `ndx_model`, `ndx_vendor_slug`
- `ndx_source` — `community`, `netbox_labs`, or `both`
- `ndx_version` — data version at last sync
- `first_imported`, `last_synced`, `update_available` (bool)
- `enrichment` — **nested, read-only** full enrichment object (or null)

Filters: `ndx_vendor_slug`, `ndx_source`, `update_available`. Search: `?q=` over manufacturer/model/`ndx_id`.

```bash
GET /api/plugins/ndx/import-records/?q=Nexus 93180YC-FX3
GET /api/plugins/ndx/import-records/?ndx_vendor_slug=cisco&update_available=true
```

### `GET enrichments/` — query by what data exists

Returns enrichment records with all category fields, the six `has_*` flags, `extensions_data`, and nested `provenance_entries`. Filter by availability flag:

```bash
GET /api/plugins/ndx/enrichments/?has_lifecycle=true
```

The enrichment object does **not** carry manufacturer/model — reach it through `import-records/` for human-readable identity.

### `POST import/` and `POST sync/`

```bash
POST /api/plugins/ndx/import/  {"ndx_ids": ["arista/dcs-7050sx3-48yc8", "cisco/nexus-93180yc-fx3"]}
```
≤5 ids import synchronously and return per-id `{success, message, created, updated}`; >5 ids queue a background job (`202`). After import the type is a normal NetBox DeviceType with component templates plus (on paid tier) enrichment.

```bash
POST /api/plugins/ndx/sync/   {}     # 202; flags update_available, does not apply changes
```

## Enrichment Data

The heart of the feature. Each device/module/rack type can have one enrichment record. Full field catalog in [references/enrichment-fields.md](references/enrichment-fields.md) — load it when reading specific attributes.

**Categories** (each with a `has_*` availability flag): lifecycle, thermal, environmental, operational, platform, snmp. A few high-value fields are promoted to top-level columns; the rich protocol detail lives in the `extensions_data` JSON blob keyed by protocol (`platform`, `snmp`, `redfish`, `gnmi`, `netconf`, `ipmi`, `bacnet`, `modbus`, `cellular`).

```jsonc
// shape of an enrichment (abbreviated)
{
  "eos_date": "2027-04", "eol_announced": "2024-10-31", "last_support_date": "2030-04-30",
  "tdp_watts": 715, "max_power_draw_watts": 1100, "max_operating_temp_c": 40.0,
  "nos": "NX-OS", "snmp_sys_object_id": "1.3.6.1.4.1.9.12.3.1.3.1234",
  "has_lifecycle": true, "has_thermal": true, "has_snmp": true,
  "extensions_data": {"snmp": {"versions": ["v2c","v3"], "mibs": [...]},
                       "gnmi": {"default_port": 6030, "encoding": ["json_ietf"]}},
  "provenance_entries": [
    {"field": "lifecycle.eos_date", "source": "https://...", "provenance_type": "vendor_eol_bulletin",
     "confidence": "high", "retrieved": "2026-01-15"}
  ]
}
```

### Rules for consuming enrichment correctly

1. **Check the `has_*` flag first.** A `True` flag with an empty value means the data **exists in the catalog but is gated by your tier** (free vs paid) — report it as "available on paid plans," not "missing."
2. **Null-check everything.** Coverage is partial; many fields are null even when the flag is true on a covered category.
3. **Lifecycle dates are strings**, format `YYYY-MM` **or** `YYYY-MM-DD`. Parse defensively.
4. **Always surface provenance + confidence.** Every non-null value has a `provenance_entries[]` entry (`source` URL, `provenance_type`, `confidence` from `high`…`unverified`). Treat `low`/`unverified` cautiously.
5. **Deep protocol data is in `extensions_data`**, not top-level columns. Only sysObjectID, OID prefix, and NOS are promoted.

### NDX lifecycle vs. core `end_of_life` (NetBox 4.7+)

NetBox 4.7 added a single optional `end_of_life` date to `DeviceType` and `ModuleType` ("the date after which this device type is no longer supported by the manufacturer"), plus `cooling_method` (`air`/`liquid`/`hybrid`/`immersion`). The two layers are complementary, not duplicates:

| | Core `end_of_life` *(4.7)* | NDX lifecycle enrichment |
|---|---|---|
| Shape | One `YYYY-MM-DD` date, no source | Six dates (`ga_date`, `eol_announced`, `eos_date`, `eosw_date`, `eosec_date`, `last_support_date`), `YYYY-MM` or `YYYY-MM-DD` |
| Provenance | None — user-maintained | `provenance_entries[]` with source URL + confidence |
| Reach | Core filters, tables, exports, any plugin, any NetBox version 4.7+ | `/api/plugins/ndx/` on Cloud/Enterprise, values on paid plans |

**NDX populates core `end_of_life` for you (NDX plugin 0.7.0+, 2026-09-19, on NetBox 4.7+).** On import and re-import, NDX writes `DeviceType`/`ModuleType.end_of_life` from the enrichment's **`last_support_date`** (the semantic match for "no longer supported by the manufacturer"), never from `eos_date` (end of sale) or `eol_announced`. Rules an agent must know:

- Controlled by the plugin setting `populate_core_end_of_life` (default **on**); off leaves lifecycle data in the enrichment record only.
- Only a strict `YYYY-MM-DD` value is written. A partial `YYYY-MM` `last_support_date` is **skipped** (logged, field left unchanged) — a guessed date is treated as worse than none.
- Local edits win: the core field is written only when it is empty or still equals the value NDX last wrote. A date a user changed is left alone on re-import.
- `cooling_method` is **not** populated. NDX thermal `cooling_type` (`air`/`liquid`/`mixed`/`passive`) stays in the enrichment record.

So on 4.7+ with a current NDX plugin, treat core `end_of_life` as NDX-maintained and do not overwrite it by hand. Write it yourself only when the setting is off, the type was imported before plugin 0.7.0 and not re-synced, or the NDX date is partial and the user accepts an expanded date:

A hand-set `end_of_life` that differs from the last NDX-written value is preserved on re-import. If you copy NDX `cooling_type` into core `cooling_method`, map `mixed` → `hybrid` and leave `passive` unset (no core equivalent) — NDX does not do this for you.

## Common Workflows

Full sequences in [references/consume-workflows.md](references/consume-workflows.md). The high-value ones:

- **Look up a known type's enrichment:** `GET import-records/?q=<model>` → read the nested `enrichment` (no second call needed) → read `eos_date`, `max_power_draw_watts`, `extensions_data.gnmi.default_port`, etc.
- **EOL/EOS exposure report:** pull deployed devices from `GET /api/dcim/devices/`, collect their `device_type` ids, match each to its NDX import record, flag types past/near `eos_date`/`last_support_date`, present with provenance. *(4.7)* Also read core `DeviceType.end_of_life` and reconcile the two (see above).
- **Populate observability config:** read `snmp_sys_object_id`, `extensions_data.snmp.mibs[]`, or `extensions_data.gnmi`/`.netconf`/`.redfish` to drive *what* to poll (NDX gives the what/how, not the config).
- **Import a catalog entry:** `POST import/` with the `ndx_id`(s).

## Access Tiers (concept level)

- **Open to the world:** the device/module/rack **type definitions** — browse the public catalog and download YAML manually, no NetBox Cloud/Enterprise required. This is the same shape as the community DTL.
- **Free** (all NetBox Cloud plans + NetBox Enterprise): the in-product NDX feature — full device-type catalog + import + enrichment **availability flags** (`has_*`), but enrichment **values** are gated.
- **Paid:** full enrichment **values**.

So a `has_lifecycle: true` with empty lifecycle fields is a tier-gating signal, not absence of data.

## Anti-Patterns

1. **Calling NDX a "plugin"** or exposing install/`PLUGINS_CONFIG` mechanics. It's a feature.
2. **Mis-stating the open/commercial boundary.** The **type definitions** (YAML) are open to the world and downloadable manually by anyone; the **enrichment values** and the **in-product NDX feature** (Cloud/Enterprise) are commercial. Don't call the whole catalog commercial, and don't imply enrichment values are free.
3. **Hitting an upstream catalog API.** Agents only call `/api/plugins/ndx/...` on their own NetBox. Do not generate code against any external NDX catalog/CDN endpoint.
4. **Reading `has_*` flags as "data present."** True + empty = exists but tier-gated. Report accordingly.
5. **Assuming completeness.** Null-check; coverage varies by vendor/age.
6. **Ignoring provenance/confidence.** Always surface `source` + `confidence`; the data is AI-assisted-curated.
7. **Treating lifecycle dates as full dates.** They may be `YYYY-MM`.
8. **Looking for manufacturer/model on the enrichment object.** They live on the import record; go through `import-records/`.
9. **Expecting deep protocol data as columns.** It's inside `extensions_data` keyed by protocol.
10. **Expecting `sync/` to update data.** Sync only flags updates; applying requires re-import.
11. **Treating NDX as instance/monitoring data.** It is device-**type** reference metadata only.

## Version Notes

### NetBox 4.7 (2026-09-02)

- Core `end_of_life` (date) and `cooling_method` on `DeviceType`/`ModuleType`; `cooling_capability`/`cooling_capacity` on Rack/RackType; full cooling models (sources, feeds, intakes, outflows). NDX enrichment stays the sourced, multi-date lifecycle and thermal record; NDX plugin 0.7.0+ writes core `end_of_life` from `last_support_date` on import (setting `populate_core_end_of_life`, default on; strict dates only; local edits preserved) and does not write `cooling_method`. NDX plugin 0.7.1 declares NetBox 4.5–4.7 compatibility.
- `GET /api/dcim/devices/` always includes `config_context` on 4.7 (`?exclude=config_context` is ignored) — keep `?fields=id,name,device_type` on the exposure-report pull.
- Selection custom fields read as `{"value", "label"}` — matters only if you store NDX-derived values in custom fields on 4.5–4.6 instances and compare them.

## References

- [references/enrichment-fields.md](references/enrichment-fields.md) — Every enrichment field by category, the `extensions_data` protocol blocks, and provenance structure. Load when reading specific attributes.
- [references/consume-workflows.md](references/consume-workflows.md) — Full API sequences for lookup, EOL exposure, observability, and import.
