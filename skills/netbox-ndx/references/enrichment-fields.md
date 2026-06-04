# NDX Enrichment Fields Reference

Field names below are the **API field names** (what you receive from `GET /api/plugins/ndx/enrichments/` or the nested `enrichment` on an import record). Every category has a `has_<category>` boolean availability flag. A flag of `true` with an empty value means the data exists in the catalog but is gated by the instance's tier.

## Lifecycle (`has_lifecycle`)

Dates are **strings**, format `YYYY-MM` or `YYYY-MM-DD`.

| Field | Meaning |
|-------|---------|
| `ga_date` | General availability |
| `eol_announced` | End-of-life announcement date |
| `eos_date` | End-of-sale date |
| `eosw_date` | End of software maintenance |
| `eosec_date` | End of security/vulnerability support |
| `last_support_date` | Last date of support/warranty |

## Thermal (`has_thermal`)

| Field | Unit | Notes |
|-------|------|-------|
| `tdp_watts` | watts | total system thermal design power |
| `airflow_cfm` | CFM | |
| `cooling_type` | — | `air`, `liquid`, `mixed`, `passive` |

## Environmental (`has_environmental`)

| Field | Unit |
|-------|------|
| `max_operating_temp_c` | °C |
| `min_operating_temp_c` | °C |
| `max_operating_humidity` | %RH |
| `min_operating_humidity` | %RH |
| `max_operating_altitude_m` | m |
| `ingress_protection` | string, e.g. `IP67`, `NEMA 4X` |
| `mtbf_hours` | hours |

## Operational (`has_operational`)

| Field | Unit |
|-------|------|
| `typical_power_draw_watts` | watts |
| `max_power_draw_watts` | watts |
| `noise_level_dba` | dBA |

## Platform (`has_platform`)

| Field | Meaning |
|-------|---------|
| `nos` | Network operating system, e.g. `EOS`, `NX-OS`, `ACOS` |
| `nos_family` | NOS family slug, e.g. `eos`, `acos` |
| `api_types` | JSON list of supported interfaces, e.g. `["cli","snmp","rest"]` |

## SNMP (`has_snmp`)

| Field | Meaning |
|-------|---------|
| `snmp_sys_object_id` | The device's SNMP `sysObjectID` OID |
| `snmp_vendor_oid_prefix` | Vendor enterprise OID prefix |

## `extensions_data` — deep protocol metadata

A JSON object keyed by protocol name. Only a thin slice of this is promoted to top-level columns; the rich detail lives here. Not every key is present for every device. Keys and notable sub-fields:

- **`platform`**: `nos`, `nos_family`, `api_types[]`, `profile`, `provenance[]`
- **`snmp`**: `sys_object_id`, `versions[]` (e.g. `v2c`, `v3`), `vendor_oid_prefix`, `mibs[]`, `profile`, `provenance[]`
- **`redfish`**: `bmc_type`, `redfish_version`, `schemas[]`, `odata_service_root`, `profile`, `provenance[]`
- **`gnmi`**: `gnmi_version`, `default_port`, `encoding[]`, `subscribe_modes[]`, `profile`, `provenance[]`
- **`netconf`**: `netconf_version`, `default_port`, `yang_models[]`, `datastores[]`, `capabilities[]`, `profile`, `provenance[]`
- **`ipmi`**: `ipmi_version`, `sensor_types[]`, `sol_capable`, `interface[]`, `provenance[]`
- **`bacnet`**: `device_type`, `protocol_revision`, `bbmd_capable`, `services[]`, `object_types[]`, `transport[]`, `provenance[]`
- **`modbus`**: `tcp_port`, `byte_order`, `unit_id`, `protocol`, `registers[]` (each: `address`, `type` [holding/input/coil/discrete], `format` [uint16/uint32/int16/int32/float32/string/bool], `count`, `scale`, `metric` [standardized `dcim_*` name], `name`, `labels`, `is_identity`, `byte_order`), `provenance[]`
- **`cellular`**: `technologies[]`, `bands_5g_nr[]`, `bands_lte[]`, `max_dl_mbps`, `max_ul_mbps`, `sim_slots`, `dual_sim`, `esim`, `modem_vendor`, `mimo`, `carrier_aggregation`, `gnss_constellations[]`, `provenance[]`

## Provenance (the trust layer)

Every non-null enrichment value carries a provenance entry, surfaced as `provenance_entries[]` on the enrichment. Per entry:

| Field | Meaning |
|-------|---------|
| `field` | dotted path of the described value, e.g. `thermal.tdp_watts`, `lifecycle.eos_date` |
| `source` | URL of the source document |
| `provenance_type` | `vendor_datasheet`, `vendor_eol_bulletin`, `vendor_product_page`, `vendor_documentation`, `partner_specs`, `third_party`, `dtl_inferred`, `family_inference`, `heuristic` |
| `confidence` | `high`, `medium-high`, `medium`, `low`, `unverified` |
| `retrieved` | date (`YYYY-MM-DD`) |
| `note` | optional free text |

Always present provenance + confidence alongside any enrichment value you surface to a user. Treat `low`/`unverified` and `heuristic`/`family_inference` types with caution and point at the `source` for verification.
