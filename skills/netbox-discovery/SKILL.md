---
name: netbox-discovery
description: >
  Configure and operate Orb Agent for automated network discovery into NetBox.
  Use when writing agent.yaml configs, setting up network/device/SNMP/worker
  discovery backends, deploying Orb Agent containers, managing policies and
  secrets, or troubleshooting discovery pipelines.
license: Apache-2.0
---

# NetBox Discovery (Orb Agent)

> **Your knowledge of Orb Agent may be outdated.** Discovery backends, configuration options, and supported platforms change between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Discovery docs | `https://netboxlabs.com/docs/discovery/` | Product overview, getting started |
| Orb Agent docs | `https://netboxlabs.com/docs/orb-agent/` | Configuration, backends, secrets |
| Config examples | `https://netboxlabs.com/docs/orb-agent/config_samples/` | Sample agent.yaml files |
| Orb Agent repo | `https://github.com/netboxlabs/orb-agent` | Source, `docs/` (authoritative key names), changelog |
| NetBox MCP server | If configured — verify discovered objects in NetBox | Post-discovery validation |

## Introduction

Orb Agent is a Docker-based network discovery agent that automatically discovers infrastructure and ingests it into NetBox via Diode. It ships five discovery backends — network (NMAP), device (NAPALM), SNMP, gNMI, and custom worker — plus observability backends (`snmp_telemetry`, `gnmi_telemetry`, pktvisor, OpenTelemetry Infinity) that export metrics over OTLP and ingest nothing into NetBox. All are configured through a single YAML file.

**Data flow:** Orb Agent → gRPC → Diode Server → Diode NetBox Plugin → NetBox

**Prerequisites:** NetBox 4.5+ (covers 4.5–4.7), Diode server deployed, diode-netbox-plugin installed in NetBox. This skill targets **orb-agent v2.16.x** (image tag `netboxlabs/orb-agent:2.16.0`).

> The Diode SDKs bundled in the agent determine which NetBox models discovery can emit. NetBox 4.6 models (CableBundle, RackGroup, VirtualMachineType) and 4.7 fields/models (cooling, `ModuleBayType`, channelized interfaces, `Service.port_mappings`) land only against a NetBox install of that version with a matching Diode plugin — see [netbox-diode](../netbox-diode/references/entity-catalog.md#netbox-47-additions).

For Diode SDK usage and custom ingestion patterns, see [netbox-diode](../netbox-diode/SKILL.md).

## Quick Reference

### Minimal agent.yaml

```yaml
orb:
  config_manager:
    active: local
  backends:
    common:
      diode:
        target: grpc://diode-server:8080/diode  # Cloud/Enterprise: https://your-instance.netboxcloud.com/diode
        client_id: ${DIODE_CLIENT_ID}
        client_secret: ${DIODE_CLIENT_SECRET}
        agent_name: my-agent
    network_discovery:    # Enable desired backends
    device_discovery:
    snmp_discovery:
  policies:
    # Backend-specific policies go here
```

### Docker Run

```bash
docker run --net=host \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:2.16.0 run -c /opt/orb/agent.yaml
```

Pin the image tag (`2.16.0`) in production; `latest` tracks the newest release.

### Dry Run (test without sending to Diode)

```yaml
backends:
  common:
    diode:
      dry_run: true             # client_id / client_secret not required in dry run
      dry_run_output_dir: /opt/orb
```

## Configuration Structure

The agent.yaml has four top-level sections under `orb:` (plus optional `labels`):

| Section | Purpose |
|---------|---------|
| `config_manager` | How policies are loaded — `active:` selects a source under `sources:` (`local`, `git`, or `fleet`) |
| `secrets_manager` | Optional external secret store — `active:` selects a provider under `sources:` |
| `backends` | Which backends to enable, plus common Diode/OTLP settings and per-backend lifecycle (`start_mode`) |
| `policies` | Per-backend discovery policy definitions (only with `config_manager.active: local`) |

Any `orb.*` key can also be overridden from the environment as `ORB_<PATH>` with `__` between path segments (e.g. `ORB_SECRETS_MANAGER__ACTIVE=vault`) — useful for injecting the secrets-manager selection at deploy time. See [references/agent-config-format.md](references/agent-config-format.md) for the complete YAML schema and override rules.

### Config Manager

Both `config_manager` and `secrets_manager` use the same shape: an `active:` key naming the source, and a `sources:` map of source configs.

- **local** — Policies defined in the same YAML file. Simplest setup.
- **git** — Polls a Git repo for policies on `schedule:`. Configured under `sources.git` with `url:`, `branch:`, `auth:` (a **string**: `basic`, `ssh`, or `github_app`), `username:`/`password:` (basic) or `private_key:` (ssh) or a `github_app:` block (`client_id`, `installation_id`, `private_key`), and `skip_tls:`. The repo needs a root `selector.yaml` that matches agents to policy files.

```yaml
config_manager:
  active: git
  sources:
    git:
      url: "https://github.com/org/policies.git"   # NOT "repo:"
      branch: main
      schedule: "*/5 * * * *"
      auth: basic                                   # string, NOT auth.type
      username: orb
      password: ${GIT_TOKEN}
      skip_tls: false
```

The agent matches selectors against its **top-level `orb.labels`** (not `backends.common.agent_labels`, which are telemetry labels only). See [references/deployment-patterns.md](references/deployment-patterns.md) for the `selector.yaml` format and GitHub App auth.

### Secrets Manager

Same `active:` + `sources:` shape. orb-agent v2.16 ships five providers:

| Provider (`active:`) | Reference syntax |
|----------------------|------------------|
| `vault` (HashiCorp KV v2) | `${vault://<mount>//<path>/<key>}` (qualified, multi-segment mounts OK), `${vault://<path>/<key>}` (needs `sources.vault.mount`), or legacy `${vault://<mount>/<path>/<key>}` — token, AppRole, UserPass, Kubernetes, LDAP auth |
| `doppler` | `${doppler://<secret_name>}` or qualified `${doppler://<project>/<config>/<secret_name>}` |
| `cyberark` (CCP, beta) | `${cyberark://<Safe>/<Object>}` or `${cyberark://<AppID>//<Safe>/<Object>/<Field>}` — `sources.cyberark.url` takes the CCP base URL or, when CCP runs under a non-default IIS app name, the full `…/api/Accounts` endpoint *(v2.16)* |
| `delinea` (Secret Server, beta) | `${delinea://id/<id>/<field>}` or `${delinea://path/<path>/<field>}` |
| `dsv` (Delinea DevOps Secrets Vault, v2.14+) | `${dsv://<secret-path>/<field-key>}` |

Optional `schedule` polls for secret rotations (auto-updates policies). Plain environment variables also work: `${VAR_NAME}` — but only for the fields each backend resolves (all `device_discovery` scope/defaults strings; SNMP credential fields only; **not** `network_discovery` scope).

## Discovery Backends

### Network Discovery (NMAP)

Scans IP ranges/subnets with NMAP and ingests discovered IP addresses (with `dns_name` from reverse lookup, VRF/tenant/role via `defaults`). `defaults.tenant` and `defaults.vrf` take a bare name or a map; a `vrf` map may carry its own `tenant` (`{name, tenant: {name, group}}`) so Diode matches a tenant-owned VRF instead of creating a tenant-less duplicate — `defaults.tenant` applies to the IPs only *(v2.16)*.

**Scope:** IPs, IP ranges, subnets, domain names.

**Key policy options:** `config.schedule`, `config.timeout` (minutes, default 5); scope `fast_mode`, `timing` (T0-T5), `ports` (list), `top_ports`, `scan_types` (syn/connect/udp/...), `os_detection`, `ping_scan`, `skip_host`, `dns_servers`, `icmp_echo`/`icmp_timestamp`/`icmp_netmask`.

**⚠️ Root required by default.** The default scan uses SYN scan (`-sS`) which requires `CAP_NET_RAW`. For rootless Podman, you must use:
```yaml
scope:
  scan_types: [connect]
  skip_host: true
  # Do NOT enable fast_mode
```

### Device Discovery (NAPALM)

Connects to devices via NAPALM (48 drivers bundled: 7 standard + 41 custom) and discovers detailed inventory — devices, interfaces, IPs, prefixes, VLANs, VRFs, modules, platforms, manufacturers, running config.

**Scope:** `hostname` (supports subnets/ranges, max 65536 addresses per policy), `username`, `password`, `driver` (optional — auto-detected among standard drivers), `optional_args`, `override_defaults`, `netbox_id`.

**Key features:**
- Behaviour toggles live under `config.options` (not `defaults`): `discovery_drivers` (opt custom drivers into auto-detect), `device_name_source: hostname|fqdn` *(v2.15)*, `emit_device_name: false` (stop proposing hostname renames — needs `netbox_id` or `asset_tag`), `discover_modules: off|linecards|full`, `discover_vrfs`, `capture_running_config`/`capture_startup_config` (+ `sanitize_config`), `create_unknown_vlans`, `emit_host_prefixes`, `emit_prefix_vlan` (`svi-name` also uses the VLAN ID the device reports for an L3 interface where the driver supports it — Junos, MikroTik, FastIron *(v2.16)*), `emit_lag_membership` (default `true`; sets `Interface.lag` from Junos aggregated-Ethernet membership — nothing is created, the `ae` aggregate must be discovered and typed `lag`) *(v2.16)*
- Nested defaults hierarchy (site, role, rack, tenant, per-entity overrides); `vlan.group` accepts a name or a map scoped by `scope_site`/`scope_site_group`/`scope_region`/`scope_location` *(v2.15)*; a configured `vrf` map (`defaults.vrf`, `ipaddress.vrf`, `prefix.vrf`, `vrf_ipv4`/`vrf_ipv6`) may name its own `tenant` — address/prefix tenants never set the VRF's tenant *(v2.16)*
- **Rack placement** *(v2.16)*: `rack` (policy `defaults` or per-target `override_defaults`) places the device in the rack of that name in its site/location; `position` (U from 1, steps of 0.5) + `face` (`front`|`rear`) are accepted **only** in a single-address target's `override_defaults`, always together, and need a rack. Quote numeric rack names (`rack: "01"`). Only the stack master is placed
- Interface pattern matching (`interface_patterns` with `match`/`type`; `interface_exclude_patterns`)
- Jumphost/SSH support via `ssh_config_file` with ProxyJump
- Custom NAPALM drivers via `INSTALL_DRIVERS_PATH` env var
- YAML anchors for credential reuse
- **Switch-stack / Virtual Chassis** — one `VirtualChassis` entity plus one `Device` per member, interfaces/IPs routed to the owning member; member names from `stack_member_name_template` (default `{name}-{id}`); StackWise Virtual supported *(v2.12)*
- `netbox_id` per-target scope option for matching an existing device by PK

### SNMP Discovery

Discovers devices, interfaces, IPs, MACs, VLANs, prefixes, VRFs, modules and switch stacks via SNMP polling.

**Scope:** `targets` (each a map with `host` — IP, subnet or range — plus optional `port`, `authentication`, `override_defaults`, `netbox_id`) with policy-level `authentication` as fallback.

**Auth:** `protocol_version: SNMPv1|SNMPv2c|SNMPv3` (the key is `protocol_version`, not `version`); v1/v2c `community`; v3 `username`, `security_level`, `auth_protocol`/`auth_passphrase` (MD5, SHA, SHA224–SHA512), `priv_protocol`/`priv_passphrase` (DES, AES, AES192/256, AES192C/256C), `context_name` *(v2.13)*.

**Key options:** `schedule`, `timeout` (seconds), `snmp_timeout`, `snmp_probe_timeout`, `retries`, `lookup_extensions_dir`; `config.options`: `discover_modules`, `discover_vrfs`, `discover_asset_tags`, `emit_prefixes`, `emit_device_name`, `interface_name_source: auto|ifname|ifdescr`, `create_unknown_vlans`, `emit_lag_membership` (default `true`; `Interface.lag` from `IEEE8023-LAG-MIB`, Junos logical units collapsed onto the physical port) *(v2.16)*, `vlan_interface_name_prefix` (rename interfaces a switch names by bare VLAN ID, e.g. `5` → `Vlan5`; changing it renames interfaces) *(v2.16)*. Defaults: `rack` (literal, never an OID) plus per-target `position`/`face`, and `vrf.tenant`, as in device discovery *(v2.16)*. Backend-level `ingest_buffer_size` bounds queued Diode ingests.

> v1/v2c subnet scans send the community string to every address in the range. Use SNMPv3 for range scans on untrusted segments.

### gNMI Discovery *(v2.12+)*

Event-driven: long-lived gNMI subscriptions (`ON_CHANGE` → `SAMPLE` → `GET` fallback) ingest device, interface, module, IP and VRF changes within seconds of them happening.

**Scope:** `targets` with `host` (single endpoint, or a CIDR/range that is swept before subscribing *(v2.15)*), `port` (default 9339), `username`/`password`, `tls` (`ca`/`cert`/`key`, `skip_verify`, `insecure`), `profile`, `mode`, `netbox_id`, `override_defaults` (the only place `position`/`face` may appear *(v2.16)*). Credentialed CIDR/range targets require verified TLS.

**Key options:** `mode` (`auto|on_change|sample|get`), `debounce_ms`, `sample_interval_ms`, `get_interval_ms`, `rescan_interval_ms`, `options.capture_config`, `options.emit_lag_membership` (default on; links a member port only to a LAG aggregate discovered in the same cycle and typed `lag`) *(v2.16)*; defaults as for device discovery (incl. scoped `vlan.group`, `rack`, and `ip_address`/`prefix`/`vlan`/`vrf` tenants as a name or `{name, group, …}` map — `vrf.tenant` is applied to every discovered VRF *(v2.16)*).

### Worker (Custom Python)

Runs custom Python packages that use the Diode Python SDK to ingest any entity type.

**Config:** `package` (required — Python package name), `schedule`.
**Scope:** Freeform (list or map — defined by the package).
**Custom packages:** Use `INSTALL_WORKERS_PATH` env var + `workers.txt`.

See [references/backend-details.md](references/backend-details.md) for full parameter reference and entity mappings.

### Backend Lifecycle *(v2.15)*

A supervisor owns backend processes. Each backend key accepts `start_mode: eager|on_demand` (on-demand backends are configured at start but launched when their first policy arrives; failed starts retry every 5 minutes) and `start_timeout` (seconds, 1–300). The image declares `snmp_telemetry` and `gnmi_telemetry` on demand by default, so a discovery-only agent runs nothing extra. Fleet-initiated resets restart backends through the same supervisor.

## Policies

Policies are defined per-backend and can run multiple named policies simultaneously (names are forwarded verbatim and must not contain `/`):

```yaml
policies:
  network_discovery:
    scan_office:
      config:
        schedule: "0 */6 * * *"
        defaults:
          tags: [discovered, office]
      scope:
        targets:
          - 10.0.0.0/24
          - 10.0.1.0/24
  device_discovery:
    switches:
      config:
        schedule: "0 2 * * *"
        defaults:
          site: main-dc
          role: switch
      scope:
        - hostname: 10.0.0.1
          username: admin
          password: ${vault://kv//network/switch_pass}
```

Omit `schedule` for run-once execution. Schedule format is standard cron.

## Deployment

### Docker (recommended)

```bash
docker run -d --name orb-agent --restart unless-stopped \
  --stop-timeout 60 --log-driver local \
  --net=host \
  -v ${PWD}:/opt/orb/ \
  --env-file ${PWD}/.env \
  netboxlabs/orb-agent:2.16.0 run -c /opt/orb/agent.yaml
```

`--net=host` is needed for NMAP raw socket scans (alternative: `-u root`). `--stop-timeout 60` lets in-flight policy runs finish; `--log-driver local` bounds log growth. Enable the runtime at boot (`systemctl enable --now docker`) so the restart policy applies after a reboot.

### Podman

- **Privileged:** `sudo podman run --privileged --net=host ...`
- **Rootless:** No sudo, but restricted to TCP connect scans only; enable `podman-restart.service` for the user and `loginctl enable-linger`.

### System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| Memory | 1.5 GB | 2 GB |
| Disk | 1 GB | 2 GB |

Docker 20.10+ (24.0+ recommended) / Podman 4.0+ (5.0+ recommended). Linux x86_64/arm64 fully supported; macOS/Windows limited (no host networking).

### Git-based Multi-Agent Management

For fleet deployments, use Git config manager with a central repo containing `selector.yaml` that matches agent labels to policy files. Agents poll on a cron schedule for config changes. GitHub App auth (`auth: github_app`) gives a short-lived, repo-scoped credential instead of a PAT.

See [references/deployment-patterns.md](references/deployment-patterns.md) for detailed deployment examples.

## Pitfalls

- **Network discovery default = SYN scan + root required.** Always test with dry run first.
- **macOS/Windows:** No `--net=host` support — network discovery is limited.
- **Rootless Podman:** Must use `scan_types: [connect]` + `skip_host: true` + no `fast_mode`.
- **Dry run first.** Validate JSON output before going live with Diode ingestion.
- **Large subnets:** NMAP scans can be slow — use `timeout` (minutes) and appropriate `timing` level.
- **NAPALM driver auto-detection** tries only the 7 standard drivers — list bundled custom drivers in `options.discovery_drivers` or set `driver` explicitly.
- **Key names matter:** `interface_patterns` entries use `match`/`type`; SNMP auth uses `protocol_version`; network ICMP flags are flat `icmp_echo`/`icmp_timestamp`/`icmp_netmask`; device-discovery IP defaults are `ipaddress`, SNMP/gNMI use `ip_address`. Unrecognized keys are warned about in the agent log *(v2.15)* — check it after changing a policy.
- **Renaming pitfalls:** switching `device_name_source` to `fqdn`, `interface_name_source`, or `vlan_interface_name_prefix` on an existing deployment creates new NetBox objects unless the existing ones are renamed first (Diode matches by name).
- **VLANs need a site and a group.** A VLAN sent without a group matches on VID alone and duplicates group-scoped VLANs; `vlan.group` without `defaults.site` scopes the group to the placeholder site `undefined`. Set both — the agent logs a warning the first time it sends VLANs it cannot match *(v2.16)*.
- **Rack placement is sticky.** `position`/`face` are re-applied every run (a device moved in NetBox moves back), a U already occupied by another device makes Diode update *that* device, and a placement NetBox rejects fails the device record for that cycle — read the Diode ingestion logs.

## Version Notes

### orb-agent v2.16.0 (2026-10-12)

- Backends shipped: device-discovery 1.34.0, snmp-discovery 1.38.0, network-discovery 1.20.0, gnmi-discovery 1.2.0, snmp-telemetry 1.1.0, gnmi-telemetry 1.1.0.
- Rack placement in device/snmp/gnmi-discovery: `rack` in `defaults` or a target's `override_defaults`; `position` + `face` per single-address target only.
- A configured VRF map names its own `tenant` (device/snmp/network-discovery; gnmi `vrf.tenant` applies to discovered VRFs). network-discovery `defaults.tenant` and gnmi address/prefix/VLAN tenants accept a `{name, group, description, comments, tags}` map.
- `Interface.lag`: device-discovery from Junos AE membership, snmp-discovery from `IEEE8023-LAG-MIB`, gnmi-discovery from OpenConfig `aggregate-id` — all behind `emit_lag_membership` (default on).
- `emit_prefix_vlan: svi-name` also resolves device-reported VLAN IDs (Junos L3 VLAN interfaces, MikroTik, FastIron; SNMP bare-number SVIs on any vendor); snmp `vlan_interface_name_prefix`.
- Agent: CyberArk `url` accepts the full CCP endpoint; warns once when VLANs are sent without a matchable scope; refuses to start a backend whose listen address another process holds; a YAML merge key beside a mapping key is now a config error.
- Telemetry backends (Observability, no NetBox ingest): one OTLP instrumentation scope per policy, carrying `policy_name`.

### orb-agent v2.15.0 (2026-09-16)

- Backend supervisor with `start_mode: on_demand` / `start_timeout`; `snmp-telemetry` v1.0.0 and `gnmi-telemetry` v1.0.0 shipped in the image (Observability — no NetBox ingest).
- Git config manager: `auth: github_app`. Heartbeats carry `last_restart_ts`; OTLP/HTTP served on the fleet telemetry bridge.
- device-discovery: `device_name_source`, scoped `vlan.group`, SVI-derived prefix↔VLAN association, unidentified IOS modules recorded, per-policy target expansion budget. gnmi-discovery: CIDR/range targets, scoped `vlan.group`. snmp-discovery: Huawei VLAN MIB, Juniper VRF MIB, `stack_member_name_template`.
- v2.14: `dsv` secrets manager. v2.13: `emit_device_name` (device), SNMPv3 `context_name`. v2.12: `gnmi_discovery` backend, `ORB_*` environment overrides, StackWise Virtual stacks, `emit_host_prefixes`, `emit_device_name` (SNMP).

## References

- [references/agent-config-format.md](references/agent-config-format.md) — Complete YAML schema for agent.yaml
- [references/backend-details.md](references/backend-details.md) — Full parameter reference and entity mappings per backend
- [references/deployment-patterns.md](references/deployment-patterns.md) — Docker, Podman, Vault, Git-managed fleet patterns

### Cross-Skill References

- [netbox-diode](../netbox-diode/SKILL.md) — Diode SDK for programmatic data ingestion (Discovery uses Diode internally)
- [netbox-assurance](../netbox-assurance/SKILL.md) — Assurance engine that compares discovered data against intended state
