# Check Reference — All 93 Built-in Checks

## Intent Checks (42)

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

### Security Intent (5)

| Check | Description | Parameters |
|-------|-------------|------------|
| `no_forbidden_values_in_context` | No forbidden config context values | `forbidden` |
| `management_vrf_enforced` | Mgmt interfaces in correct VRF | `management_vrf_name` |
| `required_context_structure` | Config context matches schema | `schema` |
| `no_plaintext_secrets_in_context` | No password patterns in context | `patterns` |
| `restricted_prefix_usage` | Prefixes restricted by role | `restricted_prefixes` |

## Config Checks (35)

*Premium tier. Requires `enable_config_engine: true`.*

| Check | Category | Description | Parameters |
|-------|----------|-------------|------------|
| `config_parse_status` | Parse | Configs parse without errors | -- |
| `config_parse_warnings` | Parse | Config parse warnings | `max_warnings: 0` |
| `undefined_references` | Parse | Undefined ACLs/route-maps | -- |
| `unused_structures` | Parse | Defined but unused structures | -- |
| `bgp_sessions` | Routing | BGP session compatibility | -- |
| `bgp_unestablished_reason` | Routing | Why BGP can't establish | -- |
| `bgp_process_config` | Routing | BGP process correctness | -- |
| `bgp_rib_validation` | Routing | Expected BGP RIB routes | `expected_routes` |
| `ospf_session_compatibility` | Routing | OSPF neighbor compatibility | -- |
| `ospf_process_config` | Routing | OSPF process/area config | -- |
| `routing_loop_detection` | Routing | Forwarding loops | -- |
| `multipath_consistency` | Routing | ECMP consistency | -- |
| `bgp_localpref_equals` | Routing | Local-pref matches expected value | `prefix`, `expected_localpref` |
| `bgp_med_equals` | Routing | MED matches expected value | `prefix`, `expected_med` |
| `route_advertised_to` | Routing | Prefix advertised to neighbors | `prefix`, `expected_neighbors` |
| `duplicate_ips` | IP | Duplicate IPs in config | -- |
| `ip_owners_conflict` | IP | IP ownership conflicts | -- |
| `layer3_topology_complete` | IP | L3 adjacency validation | -- |
| `interface_mtu_config` | IP | MTU mismatches in config | -- |
| `hsrp_vrrp_config` | IP | FHRP configuration | -- |
| `acl_reachability` | Security | Unreachable/shadowed ACL lines | -- |
| `acl_denies_traffic` | Security | ACL denies specified flows | `flow` |
| `bgp_prefix_filter_applied` | Security | eBGP peers have filters | -- |
| `bgp_authentication_configured` | Security | BGP MD5/TCP-AO auth | -- |
| `ospf_authentication_configured` | Security | OSPF message-digest auth | `require_type` |
| `management_access_acl_configured` | Security | VTY access-class ACLs | -- |
| `assert_traffic_blocked` | Security | Traffic flows blocked | `src_ip`, `dst_ip`, `applications` |
| `vxlan_vni_config` | Overlay | VXLAN VNI mapping | -- |
| `evpn_l3_vni_consistency` | Overlay | EVPN L3 VNI consistency | -- |
| `snmp_community_clients` | Overlay | SNMP community audit | -- |
| `reachability_assertion` | Reachability | Src-to-dst reachability | `src_ip`, `dst_ip`, `applications` |
| `traceroute_reachability` | Reachability | Traceroute path validation | `src_ip`, `dst_ip` |
| `route_table_completeness` | Reachability | Expected routes present | `expected_routes` |
| `differential_reachability` | Differential | Lost/new reachability (branch) | -- |
| `routing_changes` | Differential | Route changes (branch) | -- |

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

*Routing checks require config analysis engine at deployment level.*
