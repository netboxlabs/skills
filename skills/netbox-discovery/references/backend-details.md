# Backend Details Reference

Full parameter reference and entity mappings for each Orb Agent discovery backend (orb-agent v2.15.0). Key names below are verified against the backend docs in the orb-agent repo — several differ from older skill versions (`match`/`type` in interface patterns, `protocol_version` for SNMP auth, flat `icmp_*` flags, device options under `config.options`).

## Entity Mapping Summary

| Backend | Entities Created in NetBox | SDK Used | Default API port |
|---------|---------------------------|----------|------------------|
| `network_discovery` | IP Address (+ VRF/Tenant/Role via defaults; `dns_name` from reverse lookup) | Diode Go SDK | 8073 |
| `device_discovery` | Device, Interface, DeviceType, Platform, Manufacturer, Site, Role, IP Address, Prefix, VLAN, VirtualChassis, Module, ModuleBay, VRF, DeviceConfig | Diode Python SDK | 8072 |
| `snmp_discovery` | Device, Interface, IP Address, MAC Address, Platform, Manufacturer, Site, VLAN, VirtualChassis, Module, ModuleBay, Prefix, VRF | Diode Go SDK | 8070 |
| `gnmi_discovery` | Device (+ DeviceType, Platform), Interface, Module/ModuleBay, IP Address, VRF | Diode Go SDK | 8075 |
| `worker` | Any Diode entity (custom) | Diode Python SDK | 8071 |
| `snmp_telemetry`, `gnmi_telemetry` | **None** — metrics over OTLP only (Observability product) | — | 8078 / 8079 |

Every backend accepts optional `host`/`port` (and `log_level`, `log_format`) overrides under its `backends.<name>:` key.

## Network Discovery (NMAP)

### Policy Schema

```yaml
policies:
  network_discovery:
    <policy_name>:
      config:
        schedule: "0 */6 * * *"      # Cron expression (omit for run-once)
        timeout: 5                     # Minutes for the whole nmap run (default 5)
        defaults:
          description: "Discovered by NMAP"
          comments: ""
          tags: [discovered]
          vrf: mgmt                    # Optional VRF for discovered IPs
          rd: "65000:100"              # Optional; omit to match a VRF with null rd
          tenant: acme
          role: management
          network_mask: 24             # Default mask for IPv4 (default 32)
      scope:
        targets:
          - 10.0.0.0/24
          - 192.168.1.1-192.168.1.50
          - example.com
        fast_mode: false               # -F (fewer ports); must stay false for rootless
        timing: 3                      # -T0..-T5
        ports: [22, 80, 443, 500-600]  # LIST, not a string
        top_ports: 1000
        exclude_ports: [23, 9000-12000]
        max_retries: 0                 # --max-retries
        scan_types: [syn]              # syn (default, root) | connect (rootless) | udp | ack | ...
        os_detection: false            # -O
        skip_host: false               # -Pn
        ping_scan: false               # -sn; ignored when scan_types is set
        dns_servers: [8.8.8.8]
        use_target_masks: true         # default true: apply the target subnet mask to discovered IPs
        icmp_echo: true                # -PE   (flat keys — NOT a nested icmp: map)
        icmp_timestamp: false          # -PP
        icmp_netmask: false            # -PM
```

`${VAR}` substitution is **not** applied to `network_discovery` scope values — use a secrets manager reference if a value must be injected.

### Default Behavior

Without explicit options, NMAP runs: `nmap -sS -p1-1000 --open -T3 <target>` — this requires root or `CAP_NET_RAW`.

### Rootless Configuration

```yaml
scope:
  scan_types: [connect]
  skip_host: true
  ports: [22, 80, 443, 8080]   # recommended
  # fast_mode must NOT be enabled
```

## Device Discovery (NAPALM)

48 drivers ship in the image (7 standard NAPALM + 41 bundled custom drivers). Only the standard drivers are tried during auto-detection; list custom drivers in `options.discovery_drivers` or set `driver:` per target.

### Policy Schema

```yaml
policies:
  device_discovery:
    <policy_name>:
      config:
        schedule: "0 2 * * *"
        options:                       # backend behaviour toggles live HERE, not under defaults
          platform_omit_version: false
          capture_running_config: false   # ingests a DeviceConfig entity
          capture_startup_config: false
          sanitize_config: true           # redact secrets in captured configs
          port_scan_ports: [22, 23, 80, 443, 830, 57400]   # probe before discovery on ranges/subnets
          port_scan_timeout: 0.5
          discovery_drivers: [paloalto_panos, huawei_vrp]  # ordered auto-detect list (custom drivers must be listed)
          device_name_source: hostname    # hostname (default) | fqdn  (v2.15)
          emit_device_name: true          # false = never propose a hostname rename (needs netbox_id or asset_tag)
          discover_modules: off           # off | linecards | full  (Module/ModuleBay entities)
          discover_vrfs: false            # VRF entities from get_network_instances()
          create_unknown_vlans: true      # stub VLANs referenced on interfaces but absent from the VLAN DB
          emit_host_prefixes: false       # derive Prefix from /32 and /128
          emit_prefix_vlan: off           # off | svi-name
          propagate_defaults_to_prefix_scope: false
        defaults:
          site: main-dc
          role: switch
          if_type: 1000base-t            # fallback when no pattern matches (default other)
          interface_patterns:
            - match: "^Loopback"         # keys are match/type — NOT pattern/if_type
              type: virtual
          interface_exclude_patterns: ["^veth", "^docker"]
          location: rack-a1
          rack: R101
          tenant: engineering            # or a map: {name, group, description, tags}
          stack_member_name_template: "{name}-{id}"   # Virtual Chassis member naming
          description: "Discovered device"
          comments: ""
          tags: [discovered]
          # Per-entity defaults:
          device:
            model: ""                    # override discovered model
            manufacturer: ""
            platform: ""
            description: "NAPALM discovered"
            asset_tag: ""
            tags: []
          interface:
            description: ""
            tags: []
          ipaddress:                     # note: device_discovery uses `ipaddress`, snmp_discovery uses `ip_address`
            role: ""
            tenant: ""
            vrf: mgmt                    # str or {name, rd, ...}; vrf_ipv4 / vrf_ipv6 per-family overrides
            description: ""
          prefix:
            role: ""
            tenant: ""
            vrf: ""
            scope_site: ""
            scope_location: ""
          vrf:
            name: ""
            rd: ""
          vlan:
            group: campus-vlans          # bare name = scoped to defaults.site; map form below
            # group:
            #   name: "Brussels VLAN Group"
            #   scope_site_group: Brussels   # exactly one of scope_site|scope_site_group|scope_region|scope_location
            tenant: ""
            role: ""
            description: ""
      scope:
        - hostname: 10.0.0.1
          username: admin
          password: ${SWITCH_PASS}
          driver: ios                  # Optional — auto-detected
          optional_args:
            ssh_config_file: /opt/orb/ssh_config
          override_defaults:
            site: remote-dc
          netbox_id: 42                # match existing device by PK (ignored for subnets/ranges)
        - hostname: 10.0.1.0/28       # Subnet / range scanning (max 65536 addresses per policy)
          username: admin
          password: ${SWITCH_PASS}
```

`${VAR}` substitution applies to any string in `scope` and `defaults` for this backend.

### YAML Anchors for Credential Reuse

```yaml
scope:
  - hostname: 10.0.0.1
    username: &user admin
    password: &pass ${SWITCH_PASS}
  - hostname: 10.0.0.2
    username: *user
    password: *pass
```

### Jumphost / SSH Config

```yaml
optional_args:
  ssh_config_file: /opt/orb/ssh_config
```

SSH config file with ProxyJump:
```
Host 10.0.*
  ProxyJump jumphost.example.com
  User admin
  StrictHostKeyChecking no
```

### Custom NAPALM Drivers

Set `INSTALL_DRIVERS_PATH` environment variable pointing to a directory containing `drivers.txt` (one pip package per line).

## SNMP Discovery

### Policy Schema

```yaml
backends:
  snmp_discovery:
    ingest_buffer_size: 512            # backend-level: queued Diode ingest calls (default 512)

policies:
  snmp_discovery:
    <policy_name>:
      config:
        schedule: "0 */4 * * *"
        timeout: 120                   # whole policy, SECONDS (default 120)
        snmp_timeout: 5                # per SNMP operation, seconds (default 5)
        snmp_probe_timeout: 1          # reachability probe, seconds (default 1)
        retries: 0
        lookup_extensions_dir: /opt/orb/extensions
        options:
          create_unknown_vlans: true
          discover_asset_tags: false   # ENTITY-MIB entPhysicalAssetID → asset_tag (unique matcher — trust your tags)
          discover_modules: off        # off | linecards | full
          discover_vrfs: false
          emit_prefixes: true          # one Prefix per (network, VRF)
          emit_host_prefixes: false
          emit_prefix_vlan: off        # off | svi-name
          emit_device_name: true       # false = suppress sysName rename proposals (needs netbox_id or asset_tag)
          interface_name_source: auto  # auto | ifname | ifdescr (changing this renames interfaces!)
          propagate_defaults_to_prefix_scope: false
        defaults:
          site: main-dc
          location: ".1.3.6.1.2.1.1.6.0"   # literal, or an SNMP OID reference (sysLocation here)
          role: switch
          tenant: network-ops              # str or map
          tags: [snmp-discovered]
          stack_member_name_template: "{name}-{id}"
          interface_patterns:
            - match: "^(GigabitEthernet|Gi).*"
              type: 1000base-t
          interface_exclude_patterns: ["^tap.*"]
          device:
            description: ""
            model: ""                      # override auto-discovered model / manufacturer / platform
          interface:
            description: ""
            if_type: other
          ip_address:                      # note: `ip_address` here, `ipaddress` in device_discovery
            role: management
            vrf: { name: management, rd: "65000:100" }   # or bare string; vrf_ipv4 / vrf_ipv6 overrides
            description: ""
          prefix: { role: "", tenant: "", vrf: "", scope_site: "", scope_location: "" }
          vlan:
            group: datacenter-01           # bare name or map with one scope_* key (see device_discovery)
            tenant: ""
            status: ""                     # active | reserved | deprecated; default derived from the MIB
      scope:
        # Policy-level auth (fallback for targets without their own):
        authentication:
          protocol_version: SNMPv2c      # SNMPv1 | SNMPv2c | SNMPv3  (key is protocol_version, NOT version)
          community: public
        targets:
          - host: 10.0.0.0/24            # subnets and ranges probe every address first
          - host: 10.0.1.1
            port: 161
            netbox_id: 42
            override_defaults:
              role: core
          - host: 10.0.0.10
            authentication:              # per-target override replaces the policy block wholesale
              protocol_version: SNMPv3
              username: snmpuser
              security_level: authPriv   # noAuthNoPriv | authNoPriv | authPriv
              auth_protocol: SHA256      # MD5 | SHA | SHA224 | SHA256 | SHA384 | SHA512
              auth_passphrase: ${SNMP_AUTH}
              priv_protocol: AES256      # DES | AES | AES192 | AES256 | AES192C | AES256C
              priv_passphrase: ${SNMP_PRIV}
              context_name: vrf-mgmt     # SNMPv3 only (v2.13+)
```

`${VAR}` substitution applies only to `community`, `username`, `auth_passphrase`, `priv_passphrase` and `context_name`.

> **v1/v2c range scans send the community string to every address** in the range in cleartext. Prefer SNMPv3 for subnet scanning on untrusted segments, or list targets individually.

## gNMI Discovery

Event-driven: keeps long-lived gNMI subscriptions (`ON_CHANGE`, falling back to `SAMPLE` then `GET`) and ingests changes within seconds instead of on a polling schedule. Available since orb-agent v2.12; range/CIDR targets since v2.15.

```yaml
policies:
  gnmi_discovery:
    <policy_name>:
      config:
        mode: auto                     # auto | on_change | sample | get
        debounce_ms: 2000
        sample_interval_ms: 300000
        get_interval_ms: 900000
        probe_timeout_ms: 3000         # per-address sweep timeout for CIDR/range targets
        rescan_interval_ms: 0          # re-probe unsubscribed addresses; 0 = off, min 60000
        options:
          capture_config: false        # CONFIG datastore → Device.config.running
        defaults:                      # same shape as device_discovery (site, role, location, tags,
          site: main-dc                # device, interface, ip_address, vrf, vlan.group, interface_patterns, ...)
          role: switch
      scope:
        username: ${GNMI_USER}         # scope-level defaults for every target
        password: ${GNMI_PASS}
        port: 9339
        tls:
          ca: /opt/orb/ca.pem          # verified TLS is required for credentialed CIDR/range targets
        targets:
          - host: 10.0.0.11:6030       # single endpoint (inline port allowed)
            netbox_id: 42
            profile: arista_eos        # pin a vendor profile (auto-detected when omitted)
            mode: on_change
          - host: 10.0.0.0/24          # CIDR or range: swept, then each responder subscribed
            port: 9339                 # no inline :port on ranges
```

Per-target `tls` replaces the scope block entirely (`skip_verify`, `insecure`, `ca`/`cert`/`key`). A CIDR/range target may carry a password only when TLS verifies the server, unless `config.send_credentials_to_unverified_targets: true`.

## Worker (Custom Python)

### Policy Schema

```yaml
policies:
  worker:
    <policy_name>:
      config:
        package: my_discovery_package  # Required — Python package name
        schedule: "0 * * * *"
      scope:
        # Freeform — list or map, defined by the package
        targets:
          - url: https://api.example.com
            token: ${API_TOKEN}
```

Workers use the Diode Python SDK bundled in the image — check the image's SDK version before relying on NetBox 4.7 entities (see [netbox-diode](../../netbox-diode/references/entity-catalog.md#netbox-47-additions)).

### Custom Package Installation

Set `INSTALL_WORKERS_PATH` env var pointing to a directory containing `workers.txt` (one pip package per line, a `.tar.gz`, or a local project folder).

```bash
docker run --net=host \
  -v ${PWD}:/opt/orb/ \
  -v ${PWD}/workers:/opt/workers/ \
  -e INSTALL_WORKERS_PATH=/opt/workers \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:2.15.0 run -c /opt/orb/agent.yaml
```

## Telemetry Backends (Observability)

`snmp_telemetry` and `gnmi_telemetry` (both v1.0.0, shipped in the orb-agent image since v2.15.0) poll/subscribe to devices and export metrics over OTLP. They ingest **nothing into Diode/NetBox** and belong to the NetBox Observability product — this skill only notes how they coexist with discovery:

- Require `backends.common.otlp.grpc`; the agent refuses to start if a telemetry backend is enabled without it.
- The image's default config declares them `start_mode: on_demand`, so they start only when a telemetry policy arrives.
- Policy credentials may reference `${NAME}` only if the backend's `policy_env_vars` lists that name.
- `snmp_telemetry` trap reception binds the UDP port named by the policy's `listen` (conventionally 162) — publish it (`-p 162:162/udp`) when not using `--net=host`.

Policy schemas: `docs/backends/snmp_telemetry.md` and `docs/backends/gnmi_telemetry.md` in the orb-agent repo.
