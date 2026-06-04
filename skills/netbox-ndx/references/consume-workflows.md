# NDX Consume Workflows

All sequences assume an agent with a NetBox API token hitting the customer's instance. Base prefix `/api/plugins/ndx/`. NDX is read/consume — these are lookups, not writes.

## W1 — Look up enrichment for a known device type

```bash
GET /api/plugins/ndx/import-records/?q=Nexus 93180YC-FX3
```
The matched record already contains the nested `enrichment` and its `provenance_entries` — **no second call needed**. Then read attributes directly:
- `enrichment.eos_date`, `enrichment.eol_announced`, `enrichment.last_support_date`
- `enrichment.max_power_draw_watts`, `enrichment.tdp_watts`, `enrichment.max_operating_temp_c`
- `enrichment.snmp_sys_object_id`, `enrichment.extensions_data.gnmi.default_port`

Before trusting any value: check the relevant `has_*` flag. If the flag is `true` but the value is empty, the data exists in the catalog but is **tier-gated** — report "available on paid plans," not "no data."

## W2 — Find all device types that have lifecycle data

```bash
GET /api/plugins/ndx/enrichments/?has_lifecycle=true
```
Iterate results, inspect `eos_date` / `eol_announced` / `last_support_date`. The enrichment object has no manufacturer/model, so to label results, map each back via `import-records/` (the enrichment is nested there). Other flags: `has_thermal`, `has_environmental`, `has_operational`, `has_platform`, `has_snmp`.

## W3 — EOL/EOS exposure report against deployed inventory (high-value)

1. Pull deployed devices from NetBox core and collect their device-type ids:
   ```bash
   GET /api/dcim/devices/?status=active&fields=id,name,device_type&limit=1000
   ```
2. For each distinct device type, find its NDX import record (search by manufacturer + model, or match the NetBox object) and read `enrichment.eos_date` / `eol_announced` / `last_support_date`.
3. Flag devices whose type is past or near EOS/EOL. Present each finding with its provenance — filter `provenance_entries` to entries whose `field` starts with `lifecycle.` and show `source` + `confidence` so the user can verify against the vendor bulletin.
4. Null-check: not every type has lifecycle data, and on the free tier the values may be gated even when `has_lifecycle` is true.

## W4 — Populate an observability config from protocol metadata

For a device type's enrichment:
- SNMP: `snmp_sys_object_id`, `snmp_vendor_oid_prefix`, `extensions_data.snmp.mibs[]`, `extensions_data.snmp.versions[]`
- gNMI: `extensions_data.gnmi.default_port`, `.encoding[]`, `.subscribe_modes[]`
- NETCONF: `extensions_data.netconf.default_port`, `.yang_models[]`
- Redfish: `extensions_data.redfish.odata_service_root`, `.schemas[]`
- Modbus: `extensions_data.modbus.registers[]` (address/type/format/metric)

NDX tells you **what** to poll and **how**; it does not write the monitoring config. Use these as inputs to whatever observability tooling you drive.

## W5 — Import a device type from the catalog

```bash
POST /api/plugins/ndx/import/  {"ndx_ids": ["arista/dcs-7050sx3-48yc8"]}
```
- ≤5 ids: synchronous; response lists `{ndx_id, success, message, created, updated}` per id.
- >5 ids: queues a background job (`202 {"status":"queued","count":N}`); poll `import-records/` (or NetBox jobs) for completion.
- `ndx_id` format is `vendor_slug/slug`. Bad format or unknown id is reported per-id in the results.

After import, the type is a standard NetBox `DeviceType`/`ModuleType`/`RackType` with full component templates; its manufacturer is auto-created. Re-import merges/updates — it never duplicates or deletes existing components.

## W6 — Check for catalog updates

```bash
POST /api/plugins/ndx/sync/   {}     # 202 {"status":"queued"}
```
Sync **only flags** `update_available=true` on records that have a newer catalog version; it does not apply changes. To apply, re-import the affected `ndx_id`s. Find them with:
```bash
GET /api/plugins/ndx/import-records/?update_available=true
```
