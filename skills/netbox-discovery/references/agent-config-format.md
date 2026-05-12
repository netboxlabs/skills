# Agent Config Format Reference

Complete YAML schema for `agent.yaml`. All configuration lives under the top-level `orb:` key.

## Full Structure

```yaml
orb:
  config_manager:
    active: local | git
    # Git-specific options (when active: git)
    git:
      repo: https://github.com/org/policies.git
      branch: main
      auth:
        type: basic | ssh
        username: user
        password: pass
        # or ssh_key_path for SSH
      schedule: "*/5 * * * *"   # Poll interval

  secrets_manager:
    active: vault
    sources:
      vault:
        address: "https://vault.example.com:8200"
        namespace: "my-namespace"     # Optional
        timeout: 60                   # Optional
        auth: "token"                 # token | approle | userpass | kubernetes | ldap
        auth_args:
          token: "${VAULT_TOKEN}"     # For token auth
          # role_id + secret_id for AppRole
        schedule: "*/5 * * * *"       # Refresh interval
        username: xxx             # For userpass/ldap
        password: xxx
        role: xxx                 # For kubernetes
      namespace: optional/namespace
      schedule: "0 * * * *"       # Secret refresh interval

  backends:
    common:
      diode:
        target: grpc://host:8080/diode
        client_id: ${DIODE_CLIENT_ID}
        client_secret: ${DIODE_CLIENT_SECRET}
        agent_name: my-agent
        dry_run: false
        dry_run_output_dir: /opt/orb
      otlp:
        grpc: "grpc://otel-collector:4317"
      agent_labels:
        region: us-east
        env: production

    # Enable backends by listing them (no config needed):
    network_discovery:
    device_discovery:
    snmp_discovery:
    worker:

  policies:
    # See backend-details.md for policy schemas per backend
    <backend_name>:
      <policy_name>:
        config:
          schedule: "cron expression"
          timeout: 2                    # minutes (network_discovery)
          defaults: { ... }             # backend-specific
        scope:
          # backend-specific targets
```

## Secret References

Environment variables and Vault secrets can be used anywhere in the config:

```yaml
# Environment variable
password: ${MY_ENV_VAR}

# Vault secret
password: ${vault://secret/data/network/credentials/admin_pass}
```

## Git Config Manager — selector.yaml

The Git repo must contain a `selector.yaml` at the root:

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
        env: production
    policies:
      - policies/production-snmp.yaml
```

Agents are matched by their `agent_labels` to determine which policy files apply.

## Common Defaults Structure

Most backends support a nested defaults hierarchy:

```yaml
defaults:
  site: main-dc
  role: switch
  description: "Discovered by Orb Agent"
  comments: ""
  tags:
    - discovered
    - automated
  # Per-entity overrides (device_discovery, snmp_discovery):
  device:
    description: "Discovered device"
  interface:
    description: "Discovered interface"
  ipaddress:
    description: "Discovered IP"
```

## Interface Patterns

Device and SNMP discovery support regex-based interface type matching:

```yaml
defaults:
  interface_patterns:
    - pattern: "^(Ethernet|eth)"
      if_type: 1000base-t
    - pattern: "^(GigabitEthernet|ge-)"
      if_type: 1000base-t
    - pattern: "^(TenGigabitEthernet|xe-)"
      if_type: 10gbase-t
    - pattern: "^(Loopback|lo)"
      if_type: virtual
    - pattern: "^(Vlan|vlan)"
      if_type: virtual
```
