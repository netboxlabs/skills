# Backend Details Reference

Full parameter reference and entity mappings for each Orb Agent discovery backend.

## Entity Mapping Summary

| Backend | Entities Created in NetBox | SDK Used |
|---------|---------------------------|----------|
| `network_discovery` | IP Address (Global VRF) | Diode Go SDK |
| `device_discovery` | Device, Interface, DeviceType, Platform, Manufacturer, Site, Role, IP Address, Prefix, VLAN | Diode Python SDK |
| `snmp_discovery` | Device, Interface, IP Address, MAC Address, Platform, Manufacturer, Site | Diode Go SDK |
| `worker` | Any Diode entity (custom) | Diode Python SDK |

## Network Discovery (NMAP)

### Policy Schema

```yaml
policies:
  network_discovery:
    <policy_name>:
      config:
        schedule: "0 */6 * * *"      # Cron expression (omit for run-once)
        timeout: 2                     # Minutes, default 2
        defaults:
          description: "Discovered by NMAP"
          comments: ""
          tags: [discovered]
          network_mask: 24             # Default mask for IPv4
      scope:
        targets:
          - 10.0.0.0/24
          - 192.168.1.1-192.168.1.50
          - example.com
        fast_mode: false               # -F flag (fewer ports)
        timing: 3                      # T0-T5 (0=paranoid, 5=insane)
        ports: "22,80,443"             # Specific ports
        top_ports: 1000                # Scan top N ports
        exclude_ports: "9100-9200"
        scan_types:
          - syn                        # Default (requires root)
          - connect                    # TCP connect (rootless OK)
        os_detection: false
        skip_host: false               # Skip host discovery
        ping_scan: false               # -sn ping-only
        dns_servers:
          - 8.8.8.8
        use_target_masks: false
        icmp:
          echo: true
          timestamp: false
          netmask: false
```

### Default Behavior

Without explicit options, NMAP runs: `nmap -sS -p1-1000 --open -T3 <target>` — this requires root or `CAP_NET_RAW`.

### Rootless Configuration

```yaml
config:
  scan_types: [connect]
  skip_host: true
  # fast_mode must NOT be enabled
```

## Device Discovery (NAPALM)

### Policy Schema

```yaml
policies:
  device_discovery:
    <policy_name>:
      config:
        schedule: "0 2 * * *"
        defaults:
          site: main-dc
          role: switch
          if_type: 1000base-t
          interface_patterns:
            - pattern: "^Loopback"
              if_type: virtual
          location: rack-a1
          tenant: engineering
          description: "Discovered device"
          comments: ""
          tags: [discovered]
          platform_omit_version: false
          capture_running_config: false
          capture_startup_config: false
          port_scan_ports: [22, 443]
          port_scan_timeout: 5
          # Per-entity defaults:
          device:
            description: "NAPALM discovered"
          interface:
            description: ""
          ipaddress:
            description: ""
          prefix:
            description: ""
          vlan:
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
        - hostname: 10.0.1.0/28       # Subnet scanning
          username: admin
          password: ${SWITCH_PASS}
```

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
policies:
  snmp_discovery:
    <policy_name>:
      config:
        schedule: "0 */4 * * *"
        timeout: 10
        snmp_timeout: 5               # Per-device SNMP timeout (seconds)
        snmp_probe_timeout: 3
        retries: 2
        lookup_extensions_dir: /opt/orb/extensions
        defaults:
          site: main-dc
          role: switch
          tags: [snmp-discovered]
          interface_patterns:
            - pattern: "^eth"
              if_type: 1000base-t
          device:
            description: ""
          interface:
            description: ""
          ipaddress:
            description: ""
      scope:
        targets:
          - 10.0.0.0/24
          - 10.0.1.1
        # Policy-level auth (SNMPv2c):
        authentication:
          community: public
          version: 2c

        # Or SNMPv3:
        authentication:
          version: 3
          username: snmpuser
          security_level: authPriv     # noAuthNoPriv | authNoPriv | authPriv
          auth_protocol: SHA
          auth_passphrase: ${SNMP_AUTH}
          priv_protocol: AES
          priv_passphrase: ${SNMP_PRIV}

        # Per-target auth override:
        targets:
          - host: 10.0.0.1
            authentication:
              community: private
```

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
        # Freeform — defined by the package
        targets:
          - url: https://api.example.com
            token: ${API_TOKEN}
```

### Custom Package Installation

Set `INSTALL_WORKERS_PATH` env var pointing to a directory containing `workers.txt` (one pip package per line).

```bash
docker run --net=host \
  -v ${PWD}:/opt/orb/ \
  -v ${PWD}/workers:/opt/workers/ \
  -e INSTALL_WORKERS_PATH=/opt/workers \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:latest run -c /opt/orb/agent.yaml
```
