# Policy Pack Library

30 packs: 18 starter (203 rules) and 12 compliance frameworks (525 rules). Slugs are the pack file names. Starter packs are listed by default; framework packs appear only when the plugin setting `enabled_policy_packs` includes `"compliance"` (default is `["starter"]`; Premium).

## Starter Packs (18)

| Pack | Slug | Rules | Engine | Focus |
|------|------|-------|--------|-------|
| Addressing & IPAM | `addressing-ipam` | 5 | Intent | IP addressing, prefix utilization, VRF consistency |
| Cabling & Topology Audit | `cabling-topology` | 6 | Intent | Cable integrity, symmetric cabling, trace completeness |
| Data Quality | `data-quality` | 6 | Intent | Required fields, circuit terminations, config context keys |
| Naming & Standards | `naming-standards` | 8 | Intent | Naming conventions, platform consistency, VLAN ranges |
| Redundancy & Resilience | `redundancy-resilience` | 5 | Intent | Uplinks, power, BGP sessions, path redundancy |
| Security Intent Audit | `security-intent` | 5 | Intent | Forbidden values, VRF enforcement, secrets detection |
| Leaf Intent Baseline | `leaf-intent-baseline` | 10 | Intent + Config | Baseline validation for leaf switches |
| Spine Intent Baseline | `spine-intent-baseline` | 8 | Intent + Config | Baseline validation for spine switches |
| Intended Config Baseline | `config-analysis` | 19 | Config | Parse quality, model issues, BGP/OSPF, routing, ACLs, coverage |
| Pre-Change Config Validation | `config-prechange` | 10 | Config | Differential checks for branches |
| Power Resilience | `power-resilience` | 6 | Graph | Power chain completeness, redundancy, blast radius |
| Network Resilience | `network-resilience` | 9 | Graph | Topology SPOFs, failure domains, circuit diversity |
| BGP Attribute Verification | `bgp-attributes` | 3 | Config | BGP local-pref, MED, route advertisement |
| Full Resilience Audit | `full-resilience-audit` | 15 | Graph | All graph checks — power, topology, routing |
| Management Plane Security | `management-plane-security` | 36 | Config | AAA, passwords, SSH/Telnet/HTTP, VTY/console, banners, SNMP, logging, NTP |
| Layer 2 Hardening | `layer2-hardening` | 19 | Config | VLAN 1, native/trunk pruning, STP guards, DHCP snooping, DAI, port security, 802.1X |
| Routing Protocol Security | `routing-protocol-security` | 16 | Config | OSPF/BGP/EIGRP/IS-IS auth, GTSM, max-prefix, filters, uRPF, CoPP |
| Source-of-Truth Parity | `sot-parity` | 17 | Config | Hostname, interfaces, VRFs, VLANs, BGP, config-context services match NetBox |

### Configuration Compliance Starter Packs

Five starter packs evaluate **intended configuration** with the config engine (`enable_config_engine: true`; default config source = Config Templates — switch to a Data Source after install if your intended configuration lives in a repository). See [config-compliance.md](config-compliance.md).

- **Management Plane Security** (36) — central AAA with local fallback, hashed secrets and password policy, SSH-only remote access with strong algorithms, restricted and time-limited VTY/console sessions, login/MOTD banners, SNMPv3 and read-only communities, remote syslog with timestamps, authenticated NTP, configuration archiving.
- **Layer 2 Hardening** (19) — VLAN 1 not used for management, access or native trunking; explicit trunk VLAN lists; BPDU, root and loop guard; DHCP snooping, DAI, IP source guard, port security, storm control and 802.1X on access ports; unused ports shut in a parking VLAN; CDP/LLDP off on edge ports. Access/edge interface selection is a per-rule regex parameter (`access_when`, `edge`/`edge_when`).
- **Routing Protocol Security** (16) — OSPF/BGP/EIGRP/IS-IS neighbor authentication with bounded key lifetimes; eBGP TTL security, maximum-prefix and prefix filters; bogon filtering; passive-interface defaults; source routing disabled; uRPF and CoPP; plus BGP/OSPF session compatibility and duplicate router-id detection from the model layer.
- **Source-of-Truth Parity** (17) — the intended configuration says what NetBox says: hostname, interface descriptions/IPs/admin state/MTU/VRF/switchport/LAG, VRFs, VLANs, BGP neighbors and local AS, NTP/DNS/syslog/AAA/SNMP from config context. Each result reports NetBox as `expected`, configuration as `actual`.
- **Intended Config Baseline** (19; file `config-analysis.yaml`, previously "Config Analysis Baseline") — parse status/warnings, model-build issues, undefined/unused structures, BGP and OSPF compatibility, duplicate IPs, L3 topology, MTU consistency, routing loops, ECMP consistency, ACL reachability, and a coverage rule (`config_unparsed_lines`, `max_percent: 60`).

Hardening rules use vendor-neutral control names; the device's configuration family selects the syntax. A control is **skipped, not failed**, on a family that has no pattern for it yet — check `families` on `GET /checks/{name}/`.

## Compliance Framework Packs (12)

*Premium tier (`enabled_policy_packs` must include `"compliance"`).*

| Pack | Framework | Slug | Rules | Engines |
|------|-----------|------|-------|---------|
| CLOS Fabric Compliance | CLOS leaf-spine | `compliance-clos-fabric` | 19 | Intent + Config + Graph |
| TIA-942 Data Center | Data center tiers | `compliance-tia942` | 20 | Intent + Graph |
| NIS2 / DORA | EU cyber resilience | `compliance-nis2-dora` | 42 | Intent + Config + Graph |
| NIST 800-53 | US federal security | `compliance-nist-800-53` | 62 | Intent + Config + Graph |
| NERC CIP | Critical infrastructure | `compliance-nerc-cip` | 42 | Intent + Config + Graph |
| PCI-DSS | Payment card security | `compliance-pci-dss` | 46 | Intent + Config + Graph |
| MANRS | Routing security | `compliance-manrs` | 20 | Intent + Config + Graph |
| ISO 27001:2022 | Info security mgmt | `compliance-iso-27001` | 50 | Intent + Config + Graph |
| HIPAA Security Rule (2026) | US healthcare ePHI | `compliance-hipaa` | 39 | Intent + Config + Graph |
| CIS Benchmarks — Network Devices | CIS IOS XE 17.x / NX-OS / Juniper OS | `compliance-cis-network` | 64 | Config |
| DISA STIG — Routers and Switches | Cisco IOS XE Router NDM/RTR, Switch L2S | `compliance-disa-stig` | 64 | Config |
| NSA Network Infrastructure Security | NSA guide (March 2022) | `compliance-nsa-network` | 57 | Config |

**Total**: 525 rules across 12 frameworks. The regulatory packs (NIST, PCI, HIPAA, ISO 27001, NERC CIP, MANRS, NIS2/DORA) were rebuilt on the check catalog and now carry hardening controls alongside their intent and graph rules.

### Framework References on Hardening Controls

Every hardening control carries `framework_refs`, so a compliance finding can be traced to the requirement it serves. They are exposed by `GET /checks/{name}/` and shown in the UI:

| Key | Source | Example |
|-----|--------|---------|
| `cis_ios_xe` | CIS Cisco IOS XE 17.x Benchmark v2.1.0 | `2.1.1.2` |
| `cis_nxos` | CIS Cisco NX-OS Benchmark v1.0.0 | `1.2.2` |
| `cis_junos` | CIS Juniper OS Benchmark v2.0.0 | `6.10.1.2` |
| `stig` | DISA Cisco IOS XE Router NDM/RTR and Switch L2S STIG V-IDs with CAT | `V-215845 (CAT I)` |
| `nsa_2022` | NSA Network Infrastructure Security Guide section | `7.3 Utilize secure protocols` |
| `pci_dss_4` | PCI DSS v4.0 requirement | `2.2.7` |
| `nist_800_53` | NIST SP 800-53 rev. 5 control | `AC-17(2)` |

The CIS, DISA STIG and NSA packs are generated from these references; the regulatory packs cite them per rule.

## Install / Uninstall

```bash
# List all available packs (slug, name, category, rule_count, engines, installed)
curl "$NETBOX_URL/api/plugins/validation/policy-packs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# View pack details (includes full rule list)
curl "$NETBOX_URL/api/plugins/validation/policy-packs/management-plane-security/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Install
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/management-plane-security/install/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Uninstall (plugin 1.13+: allowed even when runs exist; history is kept under the policy name)
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/management-plane-security/uninstall/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

In the UI: **Validation > Policy Packs** — click Install/Uninstall.

## Customization After Installation

Installed packs create regular policies and rules. Customize freely:

### Narrow Scope

```bash
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"scope_sites": [1, 2], "scope_roles": [3]}'
```

### Choose the Config Source (config packs)

```bash
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"config_source": "datasource", "config_data_source": 3,
       "config_path_template": "{{ device.site.slug }}/{{ device.name }}.cfg"}'
```

### Adjust Thresholds

```bash
curl -X PATCH "$NETBOX_URL/api/plugins/validation/rules/42/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"parameters": {"max_minutes": 5}}'
```

### Set Triggers and Schedule

```bash
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" -H "Content-Type: application/json" \
  -d '{"trigger_on_cr_submit": true, "trigger_on_branch_merge": true, "schedule": "0 2 * * 0"}'
```

### Add or Remove Rules

Delete rules you don't need, add new ones with different checks or parameters. Deleting a rule (or a whole policy) preserves historical run results — prior runs keep the rule's name and outcome, shown as "(deleted rule)" (plugin 1.13+).

## Framework Details

### CLOS Fabric Compliance (19 rules)

**Covers**: Leaf-spine full mesh, /31 P2P links, loopback /32s, eBGP multipath, EVPN/VXLAN overlay, ECMP path redundancy, SPOF detection.

**Prerequisites**: Set `spine_role`. Config rules need a config source (Config Templates assigned, or a Data Source).

**Key controls**: `leaf_spine_connectivity`, `symmetric_cabling`, `bgp_sessions`, `evpn_l3_vni_consistency`, `device_single_point_of_failure`, `forwarding_path_redundancy`.

### TIA-942 Data Center (20 rules)

**Covers**: Power path completeness (Tier I+), N+1 feed redundancy (Tier II+), panel independence and concurrent maintainability (Tier III+), WAN circuit diversity, physical SPOF detection.

**Prerequisites**: Power feeds, panels, and power port cabling in NetBox. Circuit/provider data for WAN diversity.

### NIS2 / DORA (42 rules)

**Controls**: NIS2 Art.21(2) risk analysis, incident handling (accounting, remote syslog, timestamps, NTP), business continuity, supply chain, network security (encrypted administration, VTY restriction, SNMPv3, routing authentication), access control and authentication (AAA, lockout, hashed secrets), cryptography. DORA Art.9 ICT risk (configuration archiving), Art.10 concentration risk, Art.11 testing.

**Prerequisites**: Customize `management_vrf_name` and `restricted_prefixes`.

### NIST 800-53 (62 rules)

**Families**: SC (boundary ACLs, uRPF, bogon filters, SNMPv3, strong cryptography, CoPP), CM (incl. CM-7 least functionality), CP, AC (VTY restriction, lockout, banners, session termination, SSH-only), IA (central AAA, hashed secrets, password length, no default SNMP communities), AU (remote syslog, timestamps, command accounting, time sync).

**Prerequisites**: Customize `management_vrf_name`, `restricted_prefixes`. Hardening rules use control defaults (`password_policy_min_length.min_length: 15`, `vty_exec_timeout.max_minutes: 10`).

### NERC CIP (42 rules)

**Controls**: CIP-005 R1/R2 (ESP boundary ACLs with logged deny, encrypted interactive remote access), CIP-007 R1 (ports/services), R4 (security event monitoring), R5 (system access control), CIP-010 (change management incl. archiving), CIP-009 (recovery plans).

**Prerequisites**: Customize `management_vrf_name` for the BES management VRF and `restricted_prefixes` for ESP boundary prefixes.

### PCI-DSS (46 rules)

**Controls**: Req 1 (CDE segmentation via VRF + ACL; 1.3.1 external-interface ACLs with logged deny; 1.4.3 anti-spoofing), Req 2 (2.2.2 no default SNMP communities; 2.2.4/2.2.5 unnecessary and insecure services; 2.2.7 encrypted administrative access), Req 8 (central AAA; 15-minute idle timeout; hashed secrets; lockout; 12-character minimum), Req 10 (accounting, timestamps, remote syslog, time sync), Req 11 (segmentation testing via differential reachability).

**Prerequisites**: Replace default CDE subnet `172.16.0.0/16`. Configure `assert_traffic_blocked` for the corporate-to-CDE boundary.

### MANRS (20 rules)

**Actions**: Action 1 (filtering — BGP session validation, prefix filters, bogon filter, maximum-prefix, unique router IDs), Action 2 (anti-spoofing — duplicate IPs, uRPF, external ACLs denying private sources), Action 3 (coordination — contact docs), Action 4 (global validation — BGP authentication, TTL security, loop detection). Two placeholder rules that could never fail (`route_advertised_to` without neighbors, `bgp_rib_validation` without prefixes) were removed — add them back with your prefixes if needed.

**Prerequisites**: None site-specific.

### ISO 27001:2022 (50 rules)

**Controls**: A.8.20 (network security — SPOF/redundancy plus SSH-only administration, SNMPv3, routing authentication, CoPP, uRPF, unused ports), A.8.21, A.8.22 (segregation), A.8.5 (secure authentication), A.8.15/16/17 (logging, monitoring, clock sync), A.8.24 (cryptography), A.8.9 (config management incl. archiving), A.5.37.

**Prerequisites**: Customize `management_vrf_name` and `restricted_prefixes`.

### HIPAA Security Rule 2026 (39 rules)

**Controls**: Asset inventory & network map (§164.308(a)(1)), network segmentation / ePHI isolation (§164.312, proposed), encryption in transit (§164.312(e), §164.312(a)(2)(iv)), authentication for management and routing planes (§164.312(d) — `aaa_enabled`, `enable_secret_configured`, `login_lockout_configured`, `bgp_authentication_configured`, `vty_access_class_configured`), automatic logoff and encrypted administration (`vty_exec_timeout`, `telnet_disabled`, `ssh_version_2`, `snmp_v3_priv_required`, `fips_algorithms_only`), audit controls (§164.312(b)), contingency & 72-hour restoration (§164.308(a)(7)), change validation (§164.306(d)). Based on the 2026 NPRM (90 FR 898) — **proposed rule, final CFR numbering may change**. Design-time validation, not a compliance attestation.

**Not covered**: MFA, anti-malware, SIEM, vulnerability scanning & pen-test cadence, patch SLAs, workforce access termination, risk-analysis/IR documentation, BAAs, training, physical security.

**Prerequisites**: Set `zones[].prefixes` + ePHI `vrf` name, `restricted_prefixes`, `management_vrf_name`, and `assert_traffic_blocked` src/dst to your clinical environment.

### CIS Benchmarks — Network Devices (64 rules)

**Covers**: Every hardening control that implements a recommendation in the CIS Cisco IOS XE 17.x (v2.1.0), CIS Cisco NX-OS (v1.0.0) or CIS Juniper OS (v2.0.0) Benchmark. Rule descriptions cite recommendation numbers; severity follows the CIS profile level (Level 1 = high, Level 2 = medium).

**Not covered**: Recommendations needing runtime state, software release checks, physical controls, or features with no configuration footprint.

**Prerequisites**: None. Defaults (`min_length: 15`, `max_minutes: 10`, `min_hosts: 2`, `min_servers: 2`) match or exceed the benchmark; lower per rule if your policy differs. Controls are evaluated per family and skipped where no pattern exists.

### DISA STIG — Routers and Switches (64 rules)

**Covers**: Hardening controls mapped to the DISA Cisco IOS XE Router NDM, Router RTR and Switch L2S STIGs. Each rule cites V-IDs and CAT; severity follows the highest CAT (CAT I = critical, CAT II = high, CAT III = medium). Session timeouts 5 minutes, password length 15, remote syslog hosts 2.

**Not covered**: Runtime state, supported-release checks, PKI enrollment, MPLS/multicast specifics, DoD-specific parameters (banner rules check presence, not mandated text — add `phrases`).

**Prerequisites**: Set `login_banner_configured.phrases`; set `edge`/`external` regex parameters on interface-scoped controls to match your external interfaces.

### NSA Network Infrastructure Security (57 rules)

**Covers**: NSA *Network Infrastructure Security Guide* (March 2022): perimeter access controls (§2.4), centralized AAA (§4), local account and password hygiene (§5), remote logging and clock sync (§6), secure remote administration and services (§7), routing protections (§8), interface port hardening (§9), notification banners (§10). Each rule cites the guide section.

**Not covered**: Architecture beyond perimeter ACLs (§2.1–2.3, 2.6), security maintenance and software currency (§3), port monitoring (§9.5).

**Prerequisites**: None. Session timeouts 5 minutes and SSH retries 3 per the guide; adjust `edge`/`external` interface regexes to your topology.

## Coverage Transparency

Each framework pack documents what it covers and what it doesn't. NetBox Validation validates *intent* — the design documented in NetBox and the intended configuration — before deployment. Controls requiring runtime monitoring, human procedures, or physical security are explicitly out of scope; verifying that the live network matches intent is the job of NetBox Discovery and NetBox Assurance.
