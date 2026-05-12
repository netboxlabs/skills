# Policy Pack Library

## Starter Packs (14)

| Pack | Slug | Rules | Engine | Focus |
|------|------|-------|--------|-------|
| Addressing & IPAM | `addressing-ipam` | 5 | Intent | IP addressing, prefix utilization, VRF consistency |
| Cabling & Topology | `cabling-topology` | 6 | Intent | Cable integrity, symmetric cabling, trace completeness |
| Data Quality | `data-quality` | 6 | Intent | Required fields, circuit terminations, config context keys |
| Naming & Standards | `naming-standards` | 8 | Intent | Naming conventions, platform consistency, VLAN ranges |
| Redundancy & Resilience | `redundancy-resilience` | 5 | Intent | Uplinks, power, BGP sessions, path redundancy |
| Security Intent | `security-intent` | 5 | Intent | Forbidden values, VRF enforcement, secrets detection |
| Leaf Intent Baseline | `leaf-intent-baseline` | 10 | Intent + Config | Baseline validation for leaf switches |
| Spine Intent Baseline | `spine-intent-baseline` | 8 | Intent + Config | Baseline validation for spine switches |
| Config Analysis Baseline | `config-analysis-baseline` | 17 | Config | Parse quality, BGP, routing, ACLs, overlay |
| Pre-Change Config Validation | `pre-change-config` | 10 | Config | Differential config validation for branches |
| Power Resilience | `power-resilience` | 6 | Graph | Power chain completeness, redundancy, blast radius |
| Network Resilience | `network-resilience` | 9 | Graph | Topology SPOFs, failure domains, circuit diversity |
| BGP Attribute Verification | `bgp-attributes` | 3 | Config | BGP local-pref, MED, route advertisement |
| Full Resilience Audit | `full-resilience-audit` | 15 | Graph + Config | All graph checks — power, topology, and routing |

## Compliance Framework Packs (8)

*Premium tier.*

| Pack | Framework | Slug | Rules | Engines |
|------|-----------|------|-------|---------|
| CLOS Fabric Design | CLOS leaf-spine | `compliance-clos-fabric` | 19 | Intent + Config + Graph |
| TIA-942 Data Center | Data center tiers | `compliance-tia-942` | 20 | Intent + Graph |
| NIS2 / DORA | EU cyber resilience | `compliance-nis2-dora` | 21 | Intent + Config + Graph |
| NIST 800-53 | US federal security | `compliance-nist-800-53` | 26 | Intent + Config + Graph |
| NERC CIP | Critical infrastructure | `compliance-nerc-cip` | 18 | Intent + Config + Graph |
| PCI-DSS | Payment card security | `compliance-pci-dss` | 16 | Intent + Config + Graph |
| MANRS | Routing security | `compliance-manrs` | 15 | Intent + Config + Graph |
| ISO 27001:2022 | Info security mgmt | `compliance-iso-27001` | 22 | Intent + Config + Graph |

**Total**: 157 rules across 8 frameworks.

## Install / Uninstall

```bash
# List all available packs
curl "$NETBOX_URL/api/plugins/validation/policy-packs/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# View pack details (includes full rule list)
curl "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Install
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/install/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"

# Uninstall (only if no runs reference the policy)
curl -X POST "$NETBOX_URL/api/plugins/validation/policy-packs/addressing-ipam/uninstall/" \
  -H "Authorization: Bearer $NETBOX_TOKEN"
```

In the UI: **Validation > Policy Packs** — click Install/Uninstall.

## Customization After Installation

Installed packs create regular policies and rules. Customize freely:

### Narrow Scope

Add site, role, platform, or tag filters to the installed policy:

```bash
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"scope_sites": [1, 2], "scope_roles": [3]}'
```

### Adjust Thresholds

Edit individual rules to change parameters:

```bash
curl -X PATCH "$NETBOX_URL/api/plugins/validation/rules/42/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"parameters": {"min_uplinks": 4}}'
```

### Set Triggers and Schedule

Enable automatic validation triggers:

```bash
curl -X PATCH "$NETBOX_URL/api/plugins/validation/policies/5/" \
  -H "Authorization: Bearer $NETBOX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "trigger_on_cr_submit": true,
    "trigger_on_branch_merge": true,
    "schedule": "0 2 * * 0"
  }'
```

### Add or Remove Rules

Delete rules you don't need, add new ones with different checks or parameters.

## Framework Details

### CLOS Fabric Design (19 rules)

**Covers**: Leaf-spine full mesh, /31 P2P links, loopback /32s, eBGP multipath, EVPN/VXLAN overlay, ECMP path redundancy, SPOF detection.

**Prerequisites**: Set `spine_role` parameter. Config engine checks need Config Templates assigned.

**Key controls**: `leaf_spine_connectivity`, `symmetric_cabling`, `bgp_sessions`, `evpn_l3_vni_consistency`, `device_single_point_of_failure`, `forwarding_path_redundancy`.

### TIA-942 Data Center (20 rules)

**Covers**: Power path completeness (Tier I+), N+1 feed redundancy (Tier II+), panel independence and concurrent maintainability (Tier III+), WAN circuit diversity, physical SPOF detection.

**Prerequisites**: Power feeds, panels, and power port cabling in NetBox. Circuit/provider data for WAN diversity.

### NIS2 / DORA (21 rules)

**Controls**: NIS2 Art.21(2) risk analysis, incident handling, business continuity, supply chain security, network security. DORA Art.9-11 ICT risk, concentration risk, testing.

**Prerequisites**: Customize `management_vrf_name` and `restricted_prefixes`.

### NIST 800-53 (26 rules)

**Families**: SC (System & Comms Protection), CM (Config Management), CP (Contingency Planning), AC (Access Control).

**Prerequisites**: Customize `management_vrf_name`, `restricted_prefixes`, `ospf_authentication_configured.require_type`.

### NERC CIP (18 rules)

**Controls**: CIP-005 (Electronic Security Perimeter), CIP-007 (System Security), CIP-010 (Config Change Management), CIP-009 (Recovery Plans).

**Prerequisites**: Customize `management_vrf_name` for BES management VRF. `restricted_prefixes` for ESP boundary.

### PCI-DSS (16 rules)

**Controls**: Req 1 (CDE segmentation via VRF + ACL), Req 2 (secure config), Req 11 (segmentation testing via differential reachability).

**Prerequisites**: Replace default CDE subnet `172.16.0.0/16` with actual cardholder data environment prefix. Configure `assert_traffic_blocked` for corporate-to-CDE boundary.

### MANRS (15 rules)

**Actions**: Action 1 (filtering — BGP, prefix filters), Action 2 (anti-spoofing — duplicate IP, ACL), Action 3 (coordination — contact docs), Action 4 (global validation — BGP auth, loop detection).

**Prerequisites**: None site-specific. Works with defaults for BGP environments.

### ISO 27001:2022 (22 rules)

**Controls**: A.8.20 (network security), A.8.21 (network services), A.8.22 (segregation), A.8.24 (cryptography), A.8.9 (config management), A.5.37 (documented procedures).

**Prerequisites**: Customize `management_vrf_name` and `restricted_prefixes`.

## Coverage Transparency

Each framework pack documents what it covers and what it doesn't. NetBox Validation validates infrastructure *intent* (design documented in NetBox) before deployment. Controls requiring runtime monitoring, human procedures, or physical security are explicitly out of scope.
