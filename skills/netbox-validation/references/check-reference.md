# Check Reference — All 202 Built-in Checks

44 intent, 142 config (9 text primitives, 82 hardening controls, 17 source-of-truth parity, 34 model-layer), 16 graph. Parameters below are a summary; the authoritative schema, defaults, supported families and framework references for any check come from `GET /api/plugins/validation/checks/{name}/` — read it before writing a rule.

Checks evaluate NetBox data or intended configuration server-side; parameters are unchanged on NetBox 4.7. The 4.7 REST changes (`{value, label}` selection custom fields, `port_mappings` on services, `config_context` always present) only affect how an agent reads or fixes the objects behind a finding — see [netbox-api-integration](../../netbox-api-integration/SKILL.md).

## Intent Checks (44)

### Addressing (8)

| Check | Description | Parameters |
|-------|-------------|------------|
| `cabled_interfaces_have_ips` | Cabled interfaces have 1+ IP | `exclude_types` |
| `no_duplicate_ips` | No IP on multiple interfaces | -- |
| `management_ip_in_prefix` | Primary IP in management prefix | `prefix` |
| `ip_prefix_utilization` | Prefix utilization below threshold | `max_utilization_pct: 90` |
| `no_orphan_ips` | No IPs without assigned interface | -- |
| `loopback_has_host_route` | Loopbacks have /32 or /128 | -- |
| `point_to_point_subnet_sizing` | P2P links use /30 or /31 | `allowed_masks` |
| `ip_vrf_consistency` | VRF interface IPs from VRF prefixes | -- |

### Redundancy (7)

| Check | Description | Parameters |
|-------|-------------|------------|
| `min_cabled_uplinks` | N+ cabled uplinks | `min_uplinks: 2`, `uplink_types` |
| `min_bgp_sessions` | N+ BGP sessions | `min_sessions: 2` |
| `redundant_power` | N+ cabled power ports | `min_power_feeds: 2` |
| `site_min_devices_by_role` | N+ devices per role at site | `role`, `min_count: 2` |
| `dual_homed_circuits` | N+ circuit terminations | `min_circuits: 2` |
| `lag_min_members` | LAG has N+ members | `min_members: 2` |
| `virtual_chassis_member_count` | VC has N+ members | `min_members: 2` |

### Topology (8)

| Check | Description | Parameters |
|-------|-------------|------------|
| `leaf_spine_connectivity` | Leaf cabled to N+ spines | `min_spines: 2`, `spine_role` |
| `no_unconnected_active_interfaces` | Active ifaces cabled or uncabled type | `allowed_uncabled_types` |
| `symmetric_cabling` | Cable ends match speed/type | -- |
| `cable_trace_complete` | Cable paths traceable E2E | -- |
| `site_redundant_paths` | N+ distinct uplink paths | `min_paths: 2` |
| `rear_port_mapping_complete` | Rear ports have front port maps | -- |
| `console_connectivity` | 1+ console port connected | -- |
| `mtu_consistency_across_link` | Cable ends same MTU | -- |

### Standards (6)

| Check | Description | Parameters |
|-------|-------------|------------|
| `bgp_asn_range_consistent` | ASNs in allowed range | `min_asn`, `max_asn` |
| `interface_naming_consistent` | Interfaces follow naming pattern | `pattern_by_type` |
| `consistent_device_naming` | Devices follow site/role naming | `pattern_by_role` |
| `consistent_platform_per_role` | Same platform per role at site | -- |
| `vlan_id_range_by_site` | VLANs in allowed ID range | `min_vid`, `max_vid` |
| `prefix_role_assigned` | Active prefixes have role | -- |

### Completeness (8)

| Check | Description | Parameters |
|-------|-------------|------------|
| `config_context_required_keys` | Config context has required keys | `required_keys` |
| `circuit_terminations_complete` | Circuits have A+Z terminations | -- |
| `vlan_assignments_complete` | Tagged interfaces have VLANs | -- |
| `site_has_required_roles` | Site has 1+ device per role | `required_roles` |
| `contact_assigned_to_site` | Sites have contacts | -- |
| `custom_field_populated` | Custom fields non-empty | `custom_fields` |
| `asset_documentation_complete` | Device fields populated | `required_fields` |
| `ntp_syslog_configured` | NTP+syslog in config context | `ntp_key`, `syslog_key`, `require_both` |

### Security Intent (7)

| Check | Description | Parameters |
|-------|-------------|------------|
| `no_forbidden_values_in_context` | No forbidden config context values | `forbidden` |
| `management_vrf_enforced` | Mgmt interfaces in correct VRF | `management_vrf_name` |
| `required_context_structure` | Config context matches schema | `schema` |
| `no_plaintext_secrets_in_context` | No password patterns in context | `patterns` |
| `restricted_prefix_usage` | Prefixes restricted by role | `restricted_prefixes` |
| `secure_protocols_enforced` | No cleartext mgmt protocols (Telnet/HTTP/SNMPv1-2c/FTP) in config context | `insecure_keys`, `required_keys` |
| `ephi_zone_segmentation` | Sensitive-zone prefixes isolated in a dedicated VRF | `zones` |

## Config Checks (142)

*Premium tier. Requires `enable_config_engine: true` and a config source on the policy. Every result records `extra.layer` (`text` or `model`), `extra.family` and `extra.source`. See [config-compliance.md](config-compliance.md) for rule-writing guidance, result semantics and the skip reasons.*

### Text Primitives (9) — text layer, all families

| Check | Description | Parameters (defaults) |
|-------|-------------|------------|
| `config_contains` | Required lines/patterns present | `lines: []`, `regex: []`, `section`, `match: all`, `match_mode: exact`, `ignore_case: false` |
| `config_not_contains` | Forbidden lines/patterns absent | `lines: []`, `regex: []`, `section`, `match_mode: exact`, `ignore_case: false` |
| `config_section_present` | Top-level section exists | `section`, `match: all` |
| `config_section_absent` | Top-level section does not exist | `section` |
| `config_each_block` | Every block matching `block` satisfies require/forbid | `block`, `depth: 0`, `exclude`, `when`, `require: []`, `require_lines: []`, `forbid: []`, `forbid_lines: []`, `allow_empty: false`, `min_blocks: 0` |
| `config_value` | Capture a value and compare | `pattern`, `op: eq` (`eq\|ne\|lt\|le\|gt\|ge\|in\|not_in\|subset_of\|superset_of\|count_ge\|count_le\|regex`), `value`, `value_from: literal` (`config_context:<path>`, `netbox:<key>`), `section`, `group: 1`, `require_match: true`, `all: true` |
| `config_section_allowlist` | Everything in a section matches an allowlist | `section`, `allow: []`, `allow_lines: []`, `include_head: false` |
| `config_banner_contains` | Banner exists and contains phrases | `banner: any`, `phrases: []`, `ignore_case: true`, `min_length: 0` |
| `config_unparsed_lines` | Coverage: significant lines no rule evaluated | `max_lines: 0`, `max_percent`, `ignore: []` |

### Hardening Controls (82) — text layer, per-family pattern tables

Vendor-neutral control names; the device's family selects the syntax. Most take no parameters. Each carries `framework_refs` (`cis_ios_xe`, `cis_nxos`, `cis_junos`, `stig`, `nsa_2022`, `pci_dss_4`, `nist_800_53`). Pattern tables are per family and still expanding; a control with no pattern for the device's family returns `skip` ("no pattern for family"), never `fail` — check `families` on `GET /checks/{name}/`. Full metadata: `GET /checks/?group=Hardening`.

**Management plane (38)** — `aaa_enabled`, `aaa_login_authentication_configured`, `aaa_accounting_enabled`, `local_fallback_user_exists`, `enable_secret_configured`, `password_encryption_enabled`, `local_users_use_secret`, `password_policy_min_length` (`min_length: 15`), `login_lockout_configured`, `ssh_version_2`, `ssh_timeout_and_retries` (`max_timeout: 60`, `max_retries: 3`), `ssh_key_strength` (`min_modulus: 2048`), `telnet_disabled`, `http_server_disabled`, `https_only_management`, `vty_access_class_configured`, `vty_exec_timeout` (`max_minutes: 10`), `console_exec_timeout` (`max_minutes: 10`), `aux_port_disabled`, `login_banner_configured` (`phrases: []`), `motd_banner_configured` (`phrases: []`), `snmp_no_default_communities`, `snmp_v3_priv_required`, `snmp_community_acl_configured`, `snmp_read_only`, `snmp_trap_hosts_configured` (`min_hosts: 1`), `logging_hosts_min_count` (`min_hosts: 2`), `logging_buffered_configured`, `logging_timestamps_enabled`, `logging_source_interface_configured`, `logging_level_minimum` (`level: informational`), `ntp_servers_min_count` (`min_servers: 2`), `ntp_authentication_enabled`, `ntp_source_interface_configured`, `single_management_loopback`, `management_not_on_vlan1`, `config_archive_enabled`, `fips_algorithms_only`

**Control plane (18)** — `cdp_disabled_on_edge` (`edge`, `edge_when`), `lldp_disabled_on_edge` (`edge`, `edge_when`), `bootp_server_disabled`, `dhcp_service_disabled`, `pad_service_disabled`, `tcp_keepalives_enabled`, `ip_source_route_disabled`, `copp_configured`, `ospf_authentication_configured`, `bgp_authentication_configured`, `eigrp_authentication_configured`, `isis_authentication_configured`, `bgp_ttl_security_configured`, `bgp_max_prefix_configured`, `bgp_prefix_filter_applied`, `bogon_prefix_filter_present`, `ospf_passive_default_on_edge`, `routing_key_lifetime_max_days` (`max_days: 180`)

**Data plane (26)** — `proxy_arp_disabled`, `directed_broadcast_disabled`, `icmp_redirects_disabled`, `icmp_unreachables_disabled_external` (`external`, `external_when`), `urpf_enabled_on_edge` (`edge`, `edge_when`), `antispoof_acl_on_external` (`external`, `external_when`), `acl_denies_private_from_external`, `acl_explicit_deny_log` (`exclude`), `acl_no_permit_any_any`, `no_tunnel_interfaces`, `unused_ports_shutdown` (`unused_when`), `unused_ports_in_parking_vlan` (`vlan: 999`), `no_vlan1_on_access_ports` (`access_when`), `native_vlan_not_1`, `trunks_explicit_allowed_vlans`, `bpduguard_on_access_ports` (`access_when`), `rootguard_on_designated_ports`, `loopguard_or_udld_enabled`, `dhcp_snooping_enabled`, `dynamic_arp_inspection_enabled`, `ip_source_guard_enabled` (`access_when`), `port_security_on_access_ports` (`access_when`), `storm_control_on_access_ports` (`access_when`), `dot1x_on_access_ports` (`access_when`), `vtp_transparent_or_off`, `igmp_snooping_enabled`

### Source-of-Truth Parity (17) — `expected` = NetBox, `actual` = configuration

| Check | Layer | Description | Parameters |
|-------|-------|-------------|------------|
| `hostname_matches_netbox` | text | Hostname equals the device name | `ignore_case: true`, `strip_domain: true` |
| `interface_descriptions_match_netbox` | model | Interface descriptions match NetBox | `template`, `ignore_unmanaged: true` |
| `interface_ips_match_ipam` | model | Interface IPs match IPAM assignments | -- |
| `interface_admin_state_matches_netbox` | model | Enabled in NetBox ⇔ not `shutdown` | -- |
| `interface_mtu_matches_netbox` | model | MTU equals NetBox MTU where set | -- |
| `interface_vrf_matches_netbox` | model | Interface VRF membership matches | -- |
| `interface_switchport_matches_netbox` | model | Access/trunk mode and VLANs match | -- |
| `lag_membership_matches_netbox` | model | Channel-group membership matches LAG parents | -- |
| `vrfs_match_netbox` | model | Device VRFs match NetBox | -- |
| `vlans_match_netbox` | model | Device VLANs match site/group VLANs | -- |
| `bgp_neighbors_match_netbox` | model | BGP peers match netbox-bgp sessions | -- |
| `bgp_local_as_matches_netbox` | model | Local AS equals assigned ASN | -- |
| `ntp_servers_match_context` | model | NTP servers equal config context list | `path: ntp.servers`, `mode: equal` |
| `dns_servers_match_context` | model | DNS servers equal config context list | `path: dns.servers`, `mode: equal` |
| `syslog_hosts_match_context` | model | Logging hosts equal config context list | `path: syslog.hosts`, `mode: equal` |
| `aaa_servers_match_context` | model | TACACS/RADIUS servers equal config context | `tacacs_path: aaa.tacacs_servers`, `radius_path: aaa.radius_servers` |
| `snmp_matches_context` | model | SNMP communities/trap hosts/location/contact match | `path: snmp` |

Model-layer checks run for `ios`, `nxos`, `iosxr`, `eos`, `junos`, `panos`, `fortios`, `asa` (skip on `aoscx`). `mode` accepts `equal`, `subset`, `superset`.

### Model Layer (34) — analysis service

| Check | Group | Scope | Description | Parameters |
|-------|-------|-------|-------------|------------|
| `config_parse_status` | Config Quality | device | Every configuration parsed completely | -- |
| `config_parse_warnings` | Config Quality | device | Unrecognised lines (advisory → `warning`) | `max_warnings: 0` |
| `config_init_issues` | Config Quality | device | Issues raised while building the network model | -- |
| `undefined_references` | Config Quality | device | References to undefined ACLs/route-maps/prefix-lists | -- |
| `unused_structures` | Config Quality | device | Defined but never referenced (advisory) | -- |
| `bgp_sessions` | Routing | device | BGP sessions compatible on both ends | -- |
| `bgp_unestablished_reason` | Routing | device | Why specific BGP sessions cannot establish | -- |
| `bgp_process_config` | Routing | device | Router-ID and multipath settings | `expected_router_ids`, `multipath_ebgp` |
| `bgp_rib_validation` | Routing | device | Expected prefixes in the BGP RIB | `expected_prefixes: []` |
| `bgp_localpref_equals` | Routing | device | Local-pref for a prefix equals expected | `prefix`, `expected_localpref` |
| `bgp_med_equals` | Routing | device | MED for a prefix equals expected | `prefix`, `expected_med` |
| `route_advertised_to` | Routing | device | Prefix in RIB and sessions to each expected neighbor | `prefix`, `expected_neighbors: []` |
| `route_table_completeness` | Routing | device | Expected routes present in the main RIB | `expected_routes: []`, `node` |
| `ospf_session_compatibility` | Routing | device | OSPF adjacencies compatible | -- |
| `ospf_process_config` | Routing | device | Expected OSPF areas configured | `expected_areas: []` |
| `no_duplicate_router_ids` | Routing | fleet | No two devices share a router-id | -- |
| `duplicate_ips` | Addressing | fleet | Same IP owned by more than one interface | -- |
| `ip_owners_conflict` | Addressing | fleet | Parser-native duplicate IP detection | -- |
| `interface_mtu_config` | Topology | device | MTU matches on both ends of every L3 link | -- |
| `layer3_topology_complete` | Topology | fleet | At least one L3 adjacency modelled | -- |
| `hsrp_vrrp_config` | Redundancy | device | FHRP groups have an active router | -- |
| `vxlan_vni_config` | Overlay | device | VNIs have flood lists and local VTEP IPs | -- |
| `evpn_l3_vni_consistency` | Overlay | fleet | Each VRF maps to exactly one L3 VNI | -- |
| `routing_loop_detection` | Reachability | fleet | No forwarding loops | -- |
| `multipath_consistency` | Reachability | fleet | ECMP paths behave consistently | -- |
| `traceroute_reachability` | Reachability | fleet | Simulated traceroute src→dst accepted | `src_ip`, `dst_ip` |
| `reachability_assertion` | Reachability | fleet | Flows from source location reach destination with disposition | `dst_ip`, `src_location: "/.*/"`, `action: SUCCESS` |
| `assert_traffic_blocked` | Reachability | fleet | Traffic src→dst denied or unrouted (segmentation) | `src_ip`, `dst_ip`, `protocol`, `dst_port` |
| `acl_reachability` | Filters | device | No shadowed/unreachable filter lines | -- |
| `acl_denies_traffic` | Filters | device | A filter denies the described flow | `src_ip`, `dst_ip`, `protocol`, `dst_port` |
| `snmp_community_clients` | Management | device | Clients permitted per SNMP community | `communities: []` |
| `management_access_acl_configured` | Management | device | Legacy heuristic — prefer `vty_access_class_configured` | `management_subnets: []` |
| `differential_reachability` | Pre-change | fleet | Flows that succeed today still succeed with proposed configs (branch only) | -- |
| `routing_changes` | Pre-change | fleet | Routes removed/added by proposed configs (branch only) | -- |

Fleet-scoped checks produce one result per run (`device: null`) instead of one per device. The two pre-change checks require a branch (`requires_proposed`) and skip otherwise.

## Graph Checks (16)

*Premium tier. Requires `enable_graph_engine: true`.*

| Check | Category | Description | Parameters |
|-------|----------|-------------|------------|
| `power_path_complete` | Power | N+ complete power paths | `min_complete_paths: 1` |
| `power_redundancy` | Power | Independent feed/panel paths | `independence_level`, `min_independent_paths: 2` |
| `power_feed_capacity` | Power | Feed utilization thresholds | `warning_threshold: 80`, `critical_threshold: 95` |
| `power_three_phase_balance` | Power | Three-phase balance | `max_imbalance_percent: 20` |
| `power_feed_blast_radius` | Power | Unprotected devices per feed | `max_unprotected_devices: 0` |
| `power_panel_blast_radius` | Power | Unprotected devices per panel | `max_unprotected_devices: 0` |
| `device_single_point_of_failure` | Topology | Device removal disconnects | `spof_roles`, `min_downstream_impact: 2` |
| `cable_single_point_of_failure` | Topology | Cable removal disconnects | `cross_rack_only: true` |
| `site_connectivity_redundancy` | Infra | N+ circuit providers | `min_providers: 2`, `min_circuits: 2` |
| `circuit_path_diversity` | Infra | Circuit provider/rack diversity | `diversity_scope`, `diversity_requirements` |
| `rack_failure_impact` | Infra | Rack failure external impact | `max_external_impact: 5` |
| `shared_failure_domain` | Infra | Device pairs sharing domains | `max_shared_domains: 2`, `max_devices: 500` |
| `concurrent_maintainability` | Infra | Maintain any component w/o impact | `max_service_impact: 0` |
| `routing_convergence_impact` | Routing* | Simulate failure, prefix loss | `target_roles`, `max_targets: 10` |
| `bgp_session_criticality` | Routing* | BGP session prefix impact | `min_prefix_count: 10` |
| `forwarding_path_redundancy` | Routing* | ECMP path count | `flow_pairs`, `min_paths: 2` |

*Routing checks require the config engine's analysis service at deployment level.*
