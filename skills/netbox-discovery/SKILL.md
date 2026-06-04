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
| Orb Agent repo | `https://github.com/netboxlabs/orb-agent` | Source, Dockerfile, changelog |
| NetBox MCP server | If configured — verify discovered objects in NetBox | Post-discovery validation |

## Introduction

Orb Agent is a Docker-based network discovery agent that automatically discovers infrastructure and ingests it into NetBox via Diode. It supports four discovery backends — network (NMAP), device (NAPALM), SNMP, and custom worker — each configurable through a single YAML file.

**Data flow:** Orb Agent → gRPC → Diode Server → Diode NetBox Plugin → NetBox

**Prerequisites:** NetBox 4.5+ (covers 4.5.x–4.6.x), Diode server deployed, diode-netbox-plugin installed in NetBox. This skill targets **orb-agent v2.9.x**.

> Ingesting the NetBox 4.6 models (CableBundle, RackGroup, VirtualMachineType) — e.g. from switch-stack discovery — requires a NetBox 4.6 install with a matching Diode plugin/SDK.

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
  netboxlabs/orb-agent:latest run -c /opt/orb/agent.yaml
```

### Dry Run (test without sending to Diode)

```yaml
backends:
  common:
    diode:
      dry_run: true
      dry_run_output_dir: /opt/orb
```

## Configuration Structure

The agent.yaml has four top-level sections under `orb:`:

| Section | Purpose |
|---------|---------|
| `config_manager` | How policies are loaded — `active:` selects a source under `sources:` (`local`, `git`, or `fleet`) |
| `secrets_manager` | Optional external secret store — `active:` selects a provider under `sources:` |
| `backends` | Which discovery engines to enable, plus common Diode settings |
| `policies` | Per-backend discovery policy definitions |

See [references/agent-config-format.md](references/agent-config-format.md) for the complete YAML schema.

### Config Manager

Both `config_manager` and `secrets_manager` use the same shape: an `active:` key naming the source, and a `sources:` map of source configs.

- **local** — Policies defined in the same YAML file. Simplest setup.
- **git** — Polls a Git repo for policies. Configured under `sources.git` with `url:`, `branch:`, `auth:` (a **string**: `basic` or `ssh`), `username:`/`password:` (basic) or `private_key:` (ssh), and `skip_tls:`. The repo needs a root `selector.yaml` that matches agents to policy files.

```yaml
config_manager:
  active: git
  sources:
    git:
      url: "https://github.com/org/policies.git"   # NOT "repo:"
      branch: main
      auth: basic                                   # string, NOT auth.type
      username: orb
      password: ${GIT_TOKEN}
      skip_tls: false
```

The agent matches selectors against its **top-level `orb.labels`** (not `backends.common.agent_labels`, which are telemetry labels only). See [references/deployment-patterns.md](references/deployment-patterns.md) for the `selector.yaml` format.

### Secrets Manager

Same `active:` + `sources:` shape. orb-agent v2.9 ships four providers:

| Provider (`active:`) | Reference syntax |
|----------------------|------------------|
| `vault` (HashiCorp) | `${vault://engine/path/to/secret/key}` — token, AppRole, UserPass, Kubernetes, LDAP auth; multi-segment mount paths supported |
| `doppler` | `${doppler://<secret_name>}` or qualified `${doppler://<project>/<config>/<secret_name>}` |
| `cyberark` (CCP, beta) | `${cyberark://<Safe>/<Object>}` or `${cyberark://<AppID>//<Safe>/<Object>/<Field>}` |
| `delinea` (Secret Server, beta) | `${delinea://id/<id>/<field>}` or `${delinea://path/<path>/<field>}` |

Optional `schedule` polls for secret rotations (auto-updates policies). Plain environment variables also work: `${VAR_NAME}`.

## Discovery Backends

### Network Discovery (NMAP)

Scans IP ranges/subnets with NMAP and ingests discovered IP addresses.

**Scope:** IPs, IP ranges, subnets, domain names.

**Key policy options:** `schedule`, `fast_mode`, `timing` (T0-T5), `ports`, `top_ports`, `scan_types` (syn/connect), `os_detection`, `ping_scan`, `dns_servers`.

**⚠️ Root required by default.** The default scan uses SYN scan (`-sS`) which requires `CAP_NET_RAW`. For rootless Podman, you must use:
```yaml
config:
  scan_types: [connect]
  skip_host: true
  # Do NOT enable fast_mode
```

### Device Discovery (NAPALM)

Connects to devices via NAPALM and discovers detailed inventory — devices, interfaces, IPs, prefixes, VLANs, platforms, manufacturers.

**Scope:** `hostname` (supports subnets/ranges), `username`, `password`, `driver` (optional — auto-detected), `optional_args`.

**Key features:**
- Nested defaults hierarchy (site, role, per-entity overrides)
- Interface pattern matching (regex-based type assignment)
- Jumphost/SSH support via `ssh_config_file` with ProxyJump
- Custom NAPALM drivers via `INSTALL_DRIVERS_PATH` env var
- Config capture (`capture_running_config`, `capture_startup_config`)
- YAML anchors for credential reuse
- **Switch-stack / Virtual Chassis** — when a target is a switch stack, discovery emits one `VirtualChassis` entity plus one `Device` per member and routes interfaces/IPs to the owning member (Cisco IOS, Juniper, Aruba CX, HP Comware, Brocade FastIron, Huawei VRP)
- `netbox_id` per-target scope option for matching an existing device by PK

### SNMP Discovery

Discovers devices and interfaces via SNMP polling.

**Scope:** Hosts (IPs, subnets, ranges) with per-target or policy-level authentication.

**Auth:** SNMPv1/v2c (community string), SNMPv3 (username, security_level, auth/priv protocols and passphrases).

**Key options:** `schedule`, `timeout`, `snmp_timeout`, `retries`, `lookup_extensions_dir`.

### Worker (Custom Python)

Runs custom Python packages that use the Diode Python SDK to ingest any entity type.

**Config:** `package` (required — Python package name), `schedule`.
**Scope:** Freeform (list or map — defined by the package).
**Custom packages:** Use `INSTALL_WORKERS_PATH` env var + `workers.txt`.

See [references/backend-details.md](references/backend-details.md) for full parameter reference and entity mappings.

## Policies

Policies are defined per-backend and can run multiple named policies simultaneously:

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
          password: ${vault://kv/network/switch_pass}
```

Omit `schedule` for run-once execution. Schedule format is standard cron.

## Deployment

### Docker (recommended)

```bash
docker run --net=host \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:latest run -c /opt/orb/agent.yaml
```

`--net=host` is needed for NMAP raw socket scans. Alternative: `-u root`.

### Podman

- **Privileged:** `sudo podman run --privileged --net=host ...`
- **Rootless:** No sudo, but restricted to TCP connect scans only.

### System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| Memory | 1.5 GB | 2 GB |
| Disk | 1 GB | 2 GB |

Docker 20.10+ / Podman 4.0+. Linux x86_64/arm64 fully supported; macOS/Windows limited (no host networking).

### Git-based Multi-Agent Management

For fleet deployments, use Git config manager with a central repo containing `selector.yaml` that matches agent labels to policy files. Agents poll on a cron schedule for config changes.

See [references/deployment-patterns.md](references/deployment-patterns.md) for detailed deployment examples.

## Pitfalls

- **Network discovery default = SYN scan + root required.** Always test with dry run first.
- **macOS/Windows:** No `--net=host` support — network discovery is limited.
- **Rootless Podman:** Must use `scan_types: [connect]` + `skip_host: true` + no `fast_mode`.
- **Dry run first.** Validate JSON output before going live with Diode ingestion.
- **Large subnets:** NMAP scans can be slow — use `timeout` (minutes) and appropriate `timing` level.
- **NAPALM driver auto-detection** may fail for uncommon platforms — specify `driver` explicitly.

## References

- [references/agent-config-format.md](references/agent-config-format.md) — Complete YAML schema for agent.yaml
- [references/backend-details.md](references/backend-details.md) — Full parameter reference and entity mappings per backend
- [references/deployment-patterns.md](references/deployment-patterns.md) — Docker, Podman, Vault, Git-managed fleet patterns

### Cross-Skill References

- [netbox-diode](../netbox-diode/SKILL.md) — Diode SDK for programmatic data ingestion (Discovery uses Diode internally)
- [netbox-assurance](../netbox-assurance/SKILL.md) — Assurance engine that compares discovered data against intended state
