# Deployment Patterns Reference

## Docker — Standard Deployment

```bash
docker run --net=host \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID=your-client-id \
  -e DIODE_CLIENT_SECRET=your-client-secret \
  netboxlabs/orb-agent:latest run -c /opt/orb/agent.yaml
```

- `--net=host` is required for NMAP SYN scans (network_discovery)
- Alternative: `-u root` instead of `--net=host`
- Mount the directory containing `agent.yaml` to `/opt/orb/`

## Podman

### Privileged (full functionality)

```bash
sudo podman run --privileged --net=host \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:latest run -c /opt/orb/agent.yaml
```

### Rootless (restricted)

```bash
podman run \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:latest run -c /opt/orb/agent.yaml
```

Rootless requires network_discovery policies to use:
```yaml
config:
  scan_types: [connect]
  skip_host: true
  # Do NOT enable fast_mode
```

## Dry Run Mode

Test discovery output without sending data to Diode:

```yaml
orb:
  backends:
    common:
      diode:
        target: grpc://diode-server:8080/diode  # Cloud/Enterprise: use https://
        client_id: ${DIODE_CLIENT_ID}
        client_secret: ${DIODE_CLIENT_SECRET}
        agent_name: test-agent
        dry_run: true
        dry_run_output_dir: /opt/orb
```

JSON output files are written to the specified directory. Review these before enabling live ingestion.

## Vault Integration

```yaml
orb:
  secrets_manager:
    active: vault
    sources:
      vault:
        address: "https://vault.example.com:8200"
        auth: "approle"
        auth_args:
          role_id: "${VAULT_ROLE_ID}"
          secret_id: "${VAULT_SECRET_ID}"
        schedule: "0 * * * *"    # Refresh secrets hourly

  policies:
    device_discovery:
      switches:
        scope:
          - hostname: 10.0.0.1
            username: ${vault://kv/data/network/switch_user}
            password: ${vault://kv/data/network/switch_pass}
```

### Supported Vault Auth Methods

| Method | Key Fields |
|--------|-----------|
| `token` | `token` |
| `approle` | `role_id`, `secret_id` |
| `userpass` | `username`, `password` |
| `kubernetes` | `role`, (uses in-cluster service account) |
| `ldap` | `username`, `password` |

## Git-based Fleet Management

For managing multiple agents from a central policy repo:

### Agent Configuration

```yaml
orb:
  config_manager:
    active: git
    git:
      repo: https://github.com/org/orb-policies.git
      branch: main
      auth:
        type: basic
        username: ${GIT_USER}
        password: ${GIT_TOKEN}
      schedule: "*/5 * * * *"

  backends:
    common:
      diode:
        target: grpc://diode-server:8080/diode  # Cloud/Enterprise: use https://
        client_id: ${DIODE_CLIENT_ID}
        client_secret: ${DIODE_CLIENT_SECRET}
        agent_name: agent-us-east-01
      agent_labels:
        region: us-east
        site: dc1
        env: production

    network_discovery:
    device_discovery:
    snmp_discovery:
```

### Policy Repo — selector.yaml

```yaml
selectors:
  - match:
      labels:
        region: us-east
    policies:
      - policies/us-east-network.yaml
      - policies/us-east-devices.yaml

  - match:
      labels:
        region: eu-west
    policies:
      - policies/eu-west-network.yaml

  - match:
      labels:
        env: production
    policies:
      - policies/production-snmp.yaml
```

Each policy file contains the `policies:` section for the matched backend(s).

## System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| Memory | 1.5 GB | 2 GB |
| Disk | 1 GB | 2 GB |
| Runtime | Docker 20.10+ or Podman 4.0+ | |
| Architecture | x86_64, arm64 | |
| OS | Linux (full support) | macOS/Windows: limited (no host networking) |

## Platform Limitations

| Platform | Limitation |
|----------|-----------|
| macOS | No `--net=host` — network_discovery SYN scans unavailable |
| Windows | No `--net=host` — network_discovery SYN scans unavailable |
| Rootless Podman | TCP connect scans only, no SYN/fast_mode |
