# Agent Config Format Reference

Complete YAML schema for `agent.yaml`. All configuration lives under the top-level `orb:` key.

## Full Structure

```yaml
orb:
  labels:                  # top-level agent labels — used for git selector matching
    region: us-east
    env: production

  config_manager:
    active: local          # local | git | fleet
    sources:
      git:                 # used when active: git
        url: https://github.com/org/policies.git   # NOT "repo:"
        branch: main
        auth: basic        # a STRING: basic | ssh (NOT auth.type)
        username: user     # basic auth
        password: ${GIT_TOKEN}
        private_key: /opt/orb/id_ed25519   # ssh auth (NOT auth.ssh_key_path)
        skip_tls: false

  secrets_manager:
    active: vault          # vault | doppler | cyberark | delinea
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
      # doppler / cyberark (CCP, beta) / delinea (Secret Server, beta) are also
      # valid sources — see "Secret References" below for their reference syntaxes.

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

Environment variables and secrets-manager references can be used anywhere in the config:

```yaml
# Environment variable
password: ${MY_ENV_VAR}

# HashiCorp Vault
password: ${vault://secret/data/network/credentials/admin_pass}

# Doppler (short or qualified)
password: ${doppler://CISCO_PASSWORD}
password: ${doppler://orb/prd/CISCO_PASSWORD}

# CyberArk CCP (beta)
password: ${cyberark://Lab-DB/cisco-svc-account}
username: ${cyberark://Lab-DB/cisco-svc-account/UserName}

# Delinea Secret Server (beta)
password: ${delinea://id/42/password}
password: ${delinea://path/Servers/prod-db/password}
```

## Git Config Manager — selector.yaml

The Git repo must contain a `selector.yaml` at the root. It is a **map of named
selector blocks** — each has a `selector:` (key/value labels, matched against the
agent's top-level `orb.labels`; an empty selector matches all agents) and a
`policies:` map of named policy → file path:

```yaml
agent_selector_eu:
  selector:                 # key/value labels directly (no "labels:" wrapper)
    region: EU
  policies:
    network_policy:
      path: policies/eu-network.yaml
    snmp_policy:
      path: policies/eu-snmp.yaml
      enabled: true         # optional; set false to skip

agent_selector_all:
  selector: {}              # empty = match every agent
  policies:
    base_policy:
      path: policies/base.yaml
```

Matching is against the agent's **top-level `orb.labels`** — NOT
`backends.common.agent_labels` (those are telemetry labels applied to exported
data only).

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
