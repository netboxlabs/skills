# Configuration Compliance — Config Sources, Text Rules, Hardening Controls, Parity

The config engine evaluates **configuration compliance**: does a device's configuration satisfy the policy? Intent checks verify that NetBox *data* is right; config checks verify that the *configuration* is right — before it ever reaches the network. The rules are the same whatever the configuration came from, so a policy written against rendered templates runs unchanged against files from a repository or text submitted by a CI pipeline.

*Premium tier. Every policy that runs config rules needs `enable_config_engine: true` and a config source.*

## The Two-Step Narrative

1. **Validate intended configuration before it reaches the network.** Configuration is generated from the source of truth (NetBox Config Templates + config context) or produced by other tooling and stored in a repository. NetBox Validation evaluates that intended configuration against compliance policies — on main and on branches — so a non-compliant change is caught in review, not in production.
2. **Verify the network matches intent.** NetBox Discovery and NetBox Assurance confirm that what runs in the network matches the intended configuration and the NetBox record.

When explaining results, keep the two steps distinct: a compliance finding says the *intended* configuration violates a policy. Whether the live device differs from intent is an Assurance question.

## Config Sources

Each policy declares where its configuration text comes from (`config_source`):

| Source | `config_source` | What is evaluated | Typical use |
|---|---|---|---|
| Config Templates (default) | `template` | Each scoped device's config rendered from its NetBox Config Template (device → role → platform) with the device's config context | Intended configuration generated from NetBox |
| Data Source | `datasource` | One file per device in a synced NetBox Data Source (Git, S3, local), located by a Jinja2 path template | Intended configuration produced by Ansible, vendor tooling, IaC repos |
| Inline (API only) | `inline` (run-level, not selectable on the policy) | Configuration text submitted with the request | CI pipelines, partner tooling, ad-hoc checks |
| Collected | `collected` | Configuration collected from the network by NetBox Discovery | Not yet available — provider interface exists; policies will not need to change |

### Config Templates

Nothing to configure beyond assigning Config Templates in NetBox. Devices without a template receive `skip` results with the reason "No config template assigned (device, role or platform)" — visible, not silently omitted. On a branch run the configuration is rendered **from the branch** and evaluated as the proposed configuration; the main render is the reference for differential checks.

### Data Source

```bash
# Point a NetBox Data Source (Operations > Data Sources) at the repo, sync it, then:
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/12/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"config_source": "datasource", "config_data_source": 3,
       "config_path_template": "{{ device.site.slug }}/{{ device.name }}.cfg"}'
```

| Field | Notes |
|---|---|
| `config_data_source` | ID of the synced Data Source. Required when `config_source` is `datasource` (400 otherwise). |
| `config_path_template` | Jinja2 expression resolving a device to a file path. Default `{{ device.name }}.cfg`. Examples: `configs/{{ device.platform.slug }}/{{ device.name }}.txt` |

A device whose file is absent receives a `skip` naming the missing path. The file hash is recorded in the run's provenance. YAML import/export carries `config_source` and `config_path_template` but **not** `config_data_source` (an environment-specific ID) — PATCH it after importing.

### Inline Evaluation

```bash
curl -X POST "$NETBOX_URL/api/plugins/validation/policies/12/evaluate/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"configs": {"leaf1": "hostname leaf1\n...", "42": "hostname leaf2\n..."},
       "wait": true, "origin": "ci:build-123"}'
```

- Keys are device **names or IDs**; the devices must exist in NetBox so scoping and parity checks work. Unknown devices → 404 and nothing is created. At most 500 configs per request.
- The policy must have `enable_config_engine: true` and at least one active `engine: config` rule (400 otherwise).
- `wait: true` (default) runs synchronously and returns `{"run": {...}, "results": [...]}`. `wait: false` enqueues and returns the run with HTTP 202 — poll `GET /runs/{id}/`.
- Creates a run with `config_source: inline` and `trigger: api`. Intent and graph rules are **skipped** for inline runs. The submitted text is never stored — only results, a hash and the `origin` label.

## Configuration Families

Text-layer checks parse the configuration with a dialect for the device's **family**:

| Family | Dialect | Text layer | Model layer |
|---|---|---|---|
| `ios` | Cisco IOS / IOS-XE | yes | yes |
| `nxos` | Cisco NX-OS | yes | yes |
| `iosxr` | Cisco IOS-XR | yes | yes |
| `eos` | Arista EOS | yes | yes |
| `junos` | Juniper Junos | yes | yes |
| `panos` | Palo Alto PAN-OS | yes | yes |
| `fortios` | Fortinet FortiOS | yes | yes |
| `asa` | Cisco ASA | yes | yes |
| `aoscx` | Aruba AOS-CX | yes | no (model-layer checks skip) |

**Resolution order** (recorded per device as `resolved_by`): plugin setting `platform_families` (`setting`) → platform slug/name heuristics (`platform`) → NetBox Data Exchange `nos_family` enrichment on the device type (`ndx`) → manufacturer-gated device-type model/part-number patterns (`model`) → manufacturer default (`manufacturer`) → content sniff of the config text (`sniff`) → `unresolved`. Unresolved devices get `skip` results with reason "no config family".

Map custom platform slugs explicitly — this is authoritative and the only setting involved:

```python
PLUGINS_CONFIG = {
    "netbox_validation": {
        "platform_families": {"corp-switch": "ios", "dc-leaf": "eos", "edge-fw": "panos"},
    }
}
```

**Hardening pattern tables** are shipped per family and are still expanding — not every family has an entry for every control. A control evaluated on a family with no pattern for it returns `skip` with reason "no pattern for family" (never `fail`). Text primitives and parity checks are not affected — they work on every family the parser supports. Check `GET /checks/{name}/` → `families` before assuming a control applies to a device.

Junos and PAN-OS `set`-format output is grouped into synthetic blocks by the parser, so block-oriented primitives (`config_each_block`, `section` filters) work on it as they do on hierarchical configs.

## Writing Rules

Every config rule is `engine: config` with a `check_name` from one of four kinds. Discover names, parameters and defaults from the catalog: `GET /checks/?engine=config&layer=text`, `GET /checks/{name}/`.

### Text Primitives (9) — generic, parameterised

| Check | Purpose | Parameters (defaults) |
|---|---|---|
| `config_contains` | Required lines/patterns present | `lines: []`, `regex: []`, `section`, `match: all\|any`, `match_mode: exact\|prefix`, `ignore_case: false` |
| `config_not_contains` | Forbidden lines/patterns absent | `lines: []`, `regex: []`, `section`, `match_mode`, `ignore_case` |
| `config_section_present` | Top-level block exists | `section` (str or list), `match: all\|any` |
| `config_section_absent` | Top-level block does not exist | `section` |
| `config_each_block` | Every matching block satisfies require/forbid (for-each) | `block` (regex, required), `depth: 0`, `exclude`, `when`, `require: []`, `require_lines: []`, `forbid: []`, `forbid_lines: []`, `allow_empty: false`, `min_blocks: 0` |
| `config_value` | Extract a value with a capture group and compare | `pattern` (required), `op: eq`, `value`, `value_from: literal`, `section`, `group: 1`, `require_match: true`, `all: true` |
| `config_section_allowlist` | Everything inside a section must match an allowlist | `section`, `allow: []` (regex), `allow_lines: []`, `include_head: false` |
| `config_banner_contains` | A banner exists and contains phrases | `banner: any` (`login\|motd\|exec\|any`), `phrases: []`, `ignore_case: true`, `min_length: 0` |
| `config_unparsed_lines` | Coverage: lines no rule in the policy evaluated | `max_lines: 0`, `max_percent` (alternative), `ignore: []` |

`config_value` operators: `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `in`, `not_in`, `subset_of`, `superset_of`, `count_ge`, `count_le`, `regex`. `value_from` may be `literal` (default), `config_context:<dotted.path>` or `netbox:<key>` (engine-precomputed values such as `hostname`).

```yaml
# Required global lines
- name: Baseline services
  engine: config
  category: security
  check_name: config_contains
  severity: high
  parameters:
    lines: ["service password-encryption", "no ip http server"]

# Every VTY line restricted and SSH-only
- name: VTY hardening
  engine: config
  category: security
  check_name: config_each_block
  severity: critical
  parameters:
    block: "^line vty"
    require: ['^access-class \S+ in', '^transport input ssh$']
    forbid: ["telnet"]

# Every access port (not shut) has BPDU guard; skip uplinks
- name: Access port guards
  engine: config
  check_name: config_each_block
  parameters:
    block: "^interface (Gigabit|TenGigabit)Ethernet"
    when: "switchport mode access"
    exclude: "Uplink|Po\\d+"
    require: ["^spanning-tree bpduguard enable"]

# NTP servers must include everything in config context
- name: NTP servers from context
  engine: config
  check_name: config_value
  parameters:
    pattern: '^ntp server (\S+)'
    op: superset_of
    value_from: "config_context:ntp.servers"

# Idle timeout bounded
- name: VTY exec-timeout
  engine: config
  check_name: config_value
  parameters:
    pattern: '^ exec-timeout (\d+)'
    section: "line vty"
    op: le
    value: 10

# Coverage guard: no more than 40% of significant lines left unevaluated
- name: Coverage
  engine: config
  check_name: config_unparsed_lines
  severity: low
  parameters:
    max_percent: 40
```

### Hardening Controls (82) — vendor-neutral, by plane

A control is a check name (e.g. `ssh_version_2`) whose syntax is resolved per family through the pattern table, so the same rule applies across vendors. Most take **no parameters**; a few expose thresholds or interface selectors.

| Plane | Count | Representative controls |
|---|---|---|
| Management (38) | AAA, credentials, remote access, SNMP, logging, NTP | `aaa_enabled`, `aaa_login_authentication_configured`, `aaa_accounting_enabled`, `enable_secret_configured`, `local_users_use_secret`, `password_policy_min_length` (`min_length: 15`), `login_lockout_configured`, `ssh_version_2`, `ssh_key_strength` (`min_modulus: 2048`), `telnet_disabled`, `http_server_disabled`, `vty_access_class_configured`, `vty_exec_timeout` (`max_minutes: 10`), `login_banner_configured` (`phrases: []`), `snmp_no_default_communities`, `snmp_v3_priv_required`, `snmp_read_only`, `logging_hosts_min_count` (`min_hosts: 2`), `logging_timestamps_enabled`, `ntp_servers_min_count` (`min_servers: 2`), `ntp_authentication_enabled`, `config_archive_enabled`, `fips_algorithms_only` |
| Control (18) | Services, routing-protocol security | `cdp_disabled_on_edge`, `lldp_disabled_on_edge`, `bootp_server_disabled`, `pad_service_disabled`, `ip_source_route_disabled`, `copp_configured`, `ospf_authentication_configured`, `bgp_authentication_configured`, `eigrp_authentication_configured`, `isis_authentication_configured`, `bgp_ttl_security_configured`, `bgp_max_prefix_configured`, `bgp_prefix_filter_applied`, `bogon_prefix_filter_present`, `ospf_passive_default_on_edge`, `routing_key_lifetime_max_days` (`max_days: 180`) |
| Data (26) | Interface and Layer 2 hardening, perimeter filtering | `proxy_arp_disabled`, `icmp_redirects_disabled`, `urpf_enabled_on_edge`, `antispoof_acl_on_external`, `acl_denies_private_from_external`, `acl_explicit_deny_log`, `acl_no_permit_any_any`, `unused_ports_shutdown`, `unused_ports_in_parking_vlan` (`vlan: 999`), `no_vlan1_on_access_ports`, `native_vlan_not_1`, `trunks_explicit_allowed_vlans`, `bpduguard_on_access_ports`, `dhcp_snooping_enabled`, `dynamic_arp_inspection_enabled`, `port_security_on_access_ports`, `dot1x_on_access_ports`, `storm_control_on_access_ports` |

Full list: `GET /checks/?group=Hardening` (82 rows, each with `plane` in its description context, `parameters`, `default_params`, `families`, `framework_refs`).

**Interface-selector parameters.** Interface-scoped controls select blocks with a regex pair you can override: `edge` + `edge_when` (CDP/LLDP/uRPF), `external` + `external_when` (anti-spoofing ACLs, ICMP unreachables), `access_when` (BPDU guard, port security, 802.1X, storm control, IP source guard, VLAN 1), `unused_when` (unused ports), `exclude` (`acl_explicit_deny_log`). Defaults treat an interface as "edge/external" by description keywords (`internet|isp|wan|upstream|transit|edge|outside`) or an inbound ACL, and as "access" by `switchport mode access` without `shutdown`. Set these per rule to match your naming.

**Framework references.** Every control carries `framework_refs` keyed `cis_ios_xe`, `cis_nxos`, `cis_junos`, `stig` (V-IDs with CAT), `nsa_2022`, `pci_dss_4`, `nist_800_53`. The CIS, DISA STIG and NSA packs are generated from these references; a compliance finding on a control can be traced to the requirement it serves.

```yaml
- name: "SSH Version 2 Only"
  engine: config
  category: security
  check_name: ssh_version_2
  severity: high

- name: "Login banner with required notice"
  engine: config
  check_name: login_banner_configured
  parameters:
    phrases: ["Authorized users only"]
```

A control with a pattern that does not apply to the family (`applies: false`, e.g. no VTY concept on FortiOS) skips with reason "not applicable"; compound patterns (`all`/`any` of several primitives) report each sub-check under `extra.conditions`.

### Source-of-Truth Parity (17) — the configuration says what NetBox says

Each result reports the NetBox value as `expected` and the configuration value as `actual`. `hostname_matches_netbox` runs in the text layer; the rest run in the model layer (so they skip on `aoscx`).

| Check | Compares | Parameters |
|---|---|---|
| `hostname_matches_netbox` | hostname vs device name | `ignore_case: true`, `strip_domain: true` |
| `interface_descriptions_match_netbox` | interface descriptions | `template` (Jinja for expected description), `ignore_unmanaged: true` |
| `interface_ips_match_ipam` | interface IPs vs IPAM assignments | -- |
| `interface_admin_state_matches_netbox` | enabled vs `shutdown` | -- |
| `interface_mtu_matches_netbox` | MTU where set in NetBox | -- |
| `interface_vrf_matches_netbox` | interface VRF membership | -- |
| `interface_switchport_matches_netbox` | access/trunk mode and VLANs | -- |
| `lag_membership_matches_netbox` | channel-group vs LAG parents | -- |
| `vrfs_match_netbox` | device VRFs | -- |
| `vlans_match_netbox` | VLANs vs site/group VLANs | -- |
| `bgp_neighbors_match_netbox` | BGP peers vs netbox-bgp sessions | -- |
| `bgp_local_as_matches_netbox` | local AS vs assigned ASN | -- |
| `ntp_servers_match_context` | NTP servers vs config context | `path: ntp.servers`, `mode: equal\|subset\|superset` |
| `dns_servers_match_context` | DNS servers vs config context | `path: dns.servers`, `mode` |
| `syslog_hosts_match_context` | logging hosts vs config context | `path: syslog.hosts`, `mode` |
| `aaa_servers_match_context` | TACACS/RADIUS vs config context | `tacacs_path: aaa.tacacs_servers`, `radius_path: aaa.radius_servers` |
| `snmp_matches_context` | communities/trap hosts/location/contact | `path: snmp` |

### Model-Layer Checks (34)

The analysis service builds a vendor-independent model from the configurations and answers semantic questions: config quality (`config_parse_status`, `config_parse_warnings`, `config_init_issues`, `undefined_references`, `unused_structures`), routing (`bgp_sessions`, `ospf_session_compatibility`, `no_duplicate_router_ids`, `bgp_localpref_equals`, ...), addressing/topology/overlay (`duplicate_ips`, `interface_mtu_config`, `evpn_l3_vni_consistency`), reachability and filters (`reachability_assertion`, `assert_traffic_blocked`, `acl_reachability`, `routing_loop_detection`) and the branch-only pre-change checks (`differential_reachability`, `routing_changes`). Parameters are in [check-reference.md](check-reference.md). `bgp_authentication_configured`, `ospf_authentication_configured` and `bgp_prefix_filter_applied` are hardening controls (text layer) — the catalog resolves the name to the text-layer definition.

## Result Semantics

| Status | Meaning | Score | Finding |
|---|---|---|---|
| `pass` | Requirement satisfied | counts as passed | no |
| `fail` | Requirement violated | counts as failed | yes |
| `warning` | Advisory (parse warnings, coverage, unused structures) | counts as **passed** | no |
| `error` | The check could not run (bad regex, missing required parameter, service error) | counts as failed | yes |
| `skip` | Not evaluated (no config, family unresolved/unsupported, control not applicable, engine disabled) | **excluded** | no |

Warnings never fail a run or a device. Skips are reported on the run as `skipped_checks` and never lower a score — so a device with no configuration shows a visible gap, not a false pass or a false fail. Fleet-scoped model checks (`duplicate_ips`, `routing_loop_detection`, `assert_traffic_blocked`, `evpn_l3_vni_consistency`, ...) produce one result per run with `device: null` rather than one per device.

### Result `extra` keys

| Key | Set by | Content |
|---|---|---|
| `layer` | every config result | `text` or `model` |
| `family` | every config result | resolved family (`ios`, `junos`, ...) |
| `source` | every config result | `template`, `datasource`, `inline` |
| `reason` | skips | `no config template assigned...`, `no config family`, `no pattern for family`, `not applicable`, `family not supported` |
| `missing` | `config_contains`, `config_section_present`, `config_value` (`superset_of`), banners, hostname | required items not found |
| `extra` | `config_value` (`subset_of`), allowlists | lines/values present that are not permitted |
| `violations` | `config_each_block`, `config_section_allowlist` | per-block `{block, missing, forbidden}` (capped at 100) plus `evaluated` count |
| `mismatches` | parity checks | per-object `{name, expected, actual}` differences |
| `conditions` | compound hardening controls | one `{status, message, missing/extra/violations}` per sub-check, with `mode: all\|any` |
| `rule_parameters` | all results | snapshot of the rule's parameters at run time |

Results also carry top-level `expected`, `actual`, `message` and `remediation` (pattern tables supply vendor-specific remediation lines for failed controls).

## Provenance, Coverage, Matrix, Catalog

### `GET /runs/{id}/coverage/`

```json
{
  "run": 920,
  "config_source": "template",
  "summary": {"devices": 28, "with_config": 26, "missing": 2,
              "families": {"ios": 20, "eos": 6}, "text_rules": 31, "model_rules": 5,
              "coverage_percent": 71.4, "proposed": false},
  "devices": {
    "leaf1": {"source": "template", "identifier": "7", "origin": "Leaf Template",
              "sha256": "9f2c…", "captured_at": "2026-10-07T01:02:03Z",
              "family": "ios", "resolved_by": "platform", "lines": 412,
              "coverage": {"significant_lines": 380, "matched_lines": 271, "percent": 71.3}}
  },
  "missing": {"oob-sw1": "No config template assigned (device, role or platform)"}
}
```

Coverage is the share of significant configuration lines that at least one rule in the policy evaluated — it answers "what are we not checking?". Use it after any run with config rules to tell "the check failed" apart from "the device had no configuration to check" (`missing`, and `skip` results). Runs expose the same data as `config_source` and `config_provenance`; the run page shows it.

### `GET /runs/{id}/matrix/?engine=config`

Compact device × check status grid, designed so a client never pages through full result payloads:

```json
{"run": 920,
 "checks": [{"name": "ssh_version_2", "rule_name": "SSH Version 2 Only", "engine": "config", "severity": "high"}],
 "devices": [{"id": 12, "name": "leaf1", "cells": {"ssh_version_2": {"status": "fail", "result_id": 88121}}}],
 "fleet": {"duplicate_ips": {"status": "pass", "result_id": 88200}}}
```

`fleet` holds results without a device. `?engine=` is optional (`intent`, `config`, `graph`).

### `GET /checks/` and `GET /checks/{name}/`

Catalog of every built-in check across the three engines. Filters: `engine`, `layer` (`text`/`model`), `group` (`Text Primitives`, `Hardening`, `Source-of-Truth Parity`, `Config Quality`, `Routing`, `Reachability`, `Pre-change`, ...), `category`, `family` (keeps checks supporting the family or applying to all). Each entry: `name`, `display_name`, `description`, `engine`, `layer`, `category`, `scope` (`device`/`fleet`), `group`, `parameters` (schema), `default_params`, `families` (null = all), `sources` (null = any), `requires_proposed` (branch-only), `framework_refs`. Read the entry **before writing a rule** — it is the only reliable source for parameter names and defaults.

## CI Pipeline Pattern

```
1. Pipeline renders/collects intended configs for the changed devices
2. POST /policies/{id}/evaluate/  {"configs": {...}, "wait": true, "origin": "ci:<build>"}
3. Gate on run.status (passed | failed) and results with status=fail
4. Read result.extra.missing / violations / mismatches and result.remediation to fix
5. Re-evaluate; when green, push the change (or open the NetBox change request)
```

Because inline runs skip intent and graph rules, pair the pipeline with a scheduled or CR-triggered run of the same policy for full coverage.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Every config rule `skip`, reason "No config template assigned" | `config_source: template` and devices lack a template | Assign Config Templates (device/role/platform), or switch to a Data Source |
| `skip` naming a file path | Data Source file missing for the device | Fix `config_path_template` or add the file; re-sync the Data Source |
| `skip`, reason "no config family" | Family unresolved for the platform | Add `platform_families: {"<slug>": "<family>"}` to the plugin config |
| Hardening controls `skip`, "no pattern for family" | The control has no pattern for that family yet (tables are per family and still expanding) | Use text primitives for that family; check `GET /checks/{name}/` → `families` |
| Parity/model checks `skip`, "family not supported" | `aoscx` or unresolved family on the model layer | Text-layer rules still run; use `config_value` with `value_from: netbox:...` for parity-style rules |
| `evaluate` returns 400 | Config engine disabled or no active config rules on the policy | Set `enable_config_engine: true`; add `engine: config` rules |
| `evaluate` returns 404 | A key is not a NetBox device name/ID | Create the device first — scoping and parity need it |
| `error` result on a text rule | Invalid regex or missing required parameter (`block`, `pattern`, `section`, `lines`/`regex`) | Fix the rule; `GET /checks/{name}/` lists required parameters |
| Low `coverage_percent` | Policy evaluates few lines | Add rules, or add `config_unparsed_lines` with `ignore` for boilerplate to make the gap explicit |
