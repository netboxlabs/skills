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
5. *(NetBox 4.7+)* Include core `end_of_life` in the pull (`GET /api/dcim/device-types/?fields=id,model,manufacturer,end_of_life`) and reconcile: with NDX plugin 0.7.0+ the import already writes `end_of_life` from a full-date `last_support_date` (setting `populate_core_end_of_life`, default on), so an empty core field usually means a partial NDX date, a pre-0.7.0 import that was never re-synced, or the setting is off — those are the write-back candidates. A core date that disagrees with NDX is a user edit (NDX preserves it) — flag it with the NDX provenance so the user can decide which is right. Write-back (one PATCH per type, or a bulk PATCH to `/api/dcim/device-types/` with `[{"id": ..., "end_of_life": "YYYY-MM-DD"}]`):
   ```bash
   PATCH /api/dcim/device-types/42/   {"end_of_life": "2030-04-30"}
   ```
   `end_of_life` needs a full date — NDX deliberately skips partial `YYYY-MM` values; if you expand one (e.g. to the last day of the month) say so in the report. On 4.5–4.6 there is no core field; keep the date in the NDX record (or a custom field) only.

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

After import, the type is a standard NetBox `DeviceType`/`ModuleType`/`RackType` with full component templates; its manufacturer is auto-created. Re-import merges/updates — it never duplicates or deletes existing components. The caller needs the same `add` permission as creating the object directly (`dcim.add_devicetype`, `dcim.add_moduletype`, `dcim.add_racktype`).

Components a device-type import creates: console ports, console server ports, power ports, power outlets (linked to their inlet), interfaces (incl. bridge links, PoE, `rf_role`), front/rear ports with `positions`, module bays, device bays, and *(plugin 0.8.1+)* inventory item templates from the DTL `inventory-items` list plus front↔rear `PortTemplateMapping`s from the DTL `port-mappings` list (`front_port_position`/`rear_port_position` honoured). Module types get the same minus inventory items and bays. A mapping whose port names cannot be resolved is reported in the per-id `message`, not a failed import. Types imported before 0.8.1 pick up inventory items and port mappings on re-import.

## W6 — Check for catalog updates

```bash
POST /api/plugins/ndx/sync/   {}     # 202 {"status":"queued"}
```
Sync **only flags** `update_available=true` on records that have a newer catalog version; it does not apply changes. To apply, re-import the affected `ndx_id`s. Find them with:
```bash
GET /api/plugins/ndx/import-records/?update_available=true
```

## W7 — Merge Discovery-created duplicates of NDX device types *(plugin 0.8.0+)*

**Symptom.** An NDX-imported device type has zero devices while a second type under the same manufacturer holds the fleet. NDX writes the marketing name in `model` and the vendor PID in `part_number`; Discovery (orb-agent → Diode) pushed the same hardware with the PID as `model` and no `part_number`. NetBox is unique on `(manufacturer, model)` only, so the two never matched and the NDX enrichment applies to nothing.

**Detect (API, any tier).** For each NDX-imported type, look for a non-NDX sibling whose `model` equals its `part_number`:
```bash
GET /api/plugins/ndx/import-records/?ndx_vendor_slug=cisco&fields=ndx_manufacturer,ndx_model
GET /api/dcim/device-types/?manufacturer=cisco&fields=id,model,part_number,slug,device_count
```
A duplicate is a type with `model == <NDX type>.part_number`, no NDX import record, usually created by the `diode` user (check `GET /api/core/object-changes/?changed_object_type=dcim.devicetype&action=create&changed_object_id=<id>`).

**Fix (shell access).** The plugin ships a conservative merge command. Dry run first — it prints every candidate pair as READY or SKIP and writes nothing:
```bash
python manage.py ndx_remediate_duplicates                      # report only
python manage.py ndx_remediate_duplicates --manufacturer Cisco # narrow the scan
python manage.py ndx_remediate_duplicates --apply --user admin # merge; --user is required so the changes are changelogged
python manage.py ndx_remediate_duplicates --apply --user admin --branch my-branch   # operate inside a Branching branch
```
Options: `--created-by <username>` (default `diode` — only types created by this user are treated as duplicates; pairs whose creator cannot be confirmed are skipped) or `--any-creator` (pair on the NDX record + exact PID match alone; broader, less certain).

What a merge does, per READY pair: devices are re-pointed to the NDX type, config-context assignments are moved, then the duplicate is deleted together with its own component templates and port mappings. The NDX type's templates and enrichment are untouched. A pair is **skipped and reported** (never guessed) when the duplicate carries component templates the NDX type lacks — Discovery found interfaces the curated definition does not model, so merging would lose template coverage; the extra template names are listed. Any other reference to the duplicate also blocks that pair.

Run it only after the Discovery-side fix is in place and queued Assurance deviations are drained, otherwise the next ingest recreates the duplicates. On NetBox Cloud the command runs on the NetBox Labs side — raise a support request with the manufacturer and the pairs you found.

**Fix (no shell access).** Replicate the same rules by hand: confirm the NDX type's templates are a superset of the duplicate's, `PATCH /api/dcim/devices/<id>/ {"device_type": <ndx_type_id>}` for each device (bulk PATCH to `/api/dcim/devices/` works), move any config contexts that list the duplicate, then `DELETE /api/dcim/device-types/<dup_id>/`. Do it on a branch if Branching is available.

## W8 — Audit enrichment changes through the changelog *(plugin 0.8.0+)*

Import records, enrichments and provenance entries are change-logged. To see what a re-import or sync changed:
```bash
GET /api/core/object-changes/?changed_object_type=netbox_ndx.ndxenrichment&changed_object_id=<id>
GET /api/core/object-changes/?changed_object_type=netbox_ndx.ndxprovenance
```
A provenance change renders `changed_object` as `{id, field, source, provenance_type, confidence, retrieved, note}` — no `url`, because provenance has no endpoint of its own; read the current entries through the parent enrichment. On plugins before 0.8.0 any changelog page that contained a provenance change returned HTTP 500 — if an unfiltered `object-changes/` call 500s on an older instance, exclude that type or upgrade.
