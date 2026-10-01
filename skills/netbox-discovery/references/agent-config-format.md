# Agent Config Format Reference

Complete YAML schema for `agent.yaml` (orb-agent v2.15.0). All configuration lives under the top-level `orb:` key.

## Full Structure

```yaml
version: 1.0               # optional, informational
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
        schedule: "*/5 * * * *"   # poll for changes; omit = fetch once at startup
        auth: basic        # a STRING: basic | ssh | github_app (omit for public repos)
        username: user     # basic auth
        password: ${GIT_TOKEN}     # basic: token/password; ssh: key passphrase
        private_key: /opt/orb/id_ed25519   # ssh auth (NOT auth.ssh_key_path)
        skip_tls: false
        github_app:        # only with auth: github_app (github.com only, v2.15+)
          client_id: "Iv23liAbCdEfGhIjKlMn"     # App Client ID (numeric App ID also accepted)
          installation_id: "78901234"           # numeric installation id — NOT the App ID
          private_key: /opt/orb/github-app.pem  # path, or the PEM content itself (e.g. ${GITHUB_APP_KEY})

  secrets_manager:
    active: vault          # vault | doppler | cyberark | delinea | dsv
    sources:
      vault:
        address: "https://vault.example.com:8200"
        namespace: "my-namespace"     # Optional
        mount: "kv"                   # Optional default KV-v2 mount → enables short form ${vault://path/key}
        timeout: 60                   # Optional
        auth: "token"                 # token | approle | userpass | kubernetes | ldap
        auth_args:
          token: "${VAULT_TOKEN}"     # token auth
          # role_id + secret_id       # approle
          # username + password       # userpass / ldap
          # role                      # kubernetes (pod service account)
        schedule: "*/5 * * * *"       # Refresh interval
      doppler:
        token: ${DOPPLER_TOKEN}
        project: orb
        config: prd
        schedule: "*/5 * * * *"
      cyberark:                       # CCP (beta)
        url: https://ccp.corp.example.com
        app_id: orb-agent
        client_cert: /opt/orb/secrets/orb.crt
        client_key: /opt/orb/secrets/orb.key
        schedule: "*/5 * * * *"
      dsv:                            # Delinea DevOps Secrets Vault (v2.14+)
        tenant: acme                  # acme.secretsvaultcloud.com
        client_id: ${DSV_CLIENT_ID}
        client_secret: ${DSV_CLIENT_SECRET}
        tld: com                      # Optional: com | eu | com.au ...
        schedule: "*/5 * * * *"       # Optional; omit = fetch once on first reference
      # delinea (Secret Server, beta) — see "Secret References" below

  backends:
    common:
      diode:
        target: grpc://host:8080/diode
        client_id: ${DIODE_CLIENT_ID}     # not required when dry_run: true
        client_secret: ${DIODE_CLIENT_SECRET}
        agent_name: my-agent
        dry_run: false
        dry_run_output_dir: /opt/orb
      otlp:
        grpc: "grpc://otel-collector:4317"   # required by snmp_telemetry / gnmi_telemetry
        http: "http://otel-collector:4318"   # used by pktvisor
        agent_labels:                        # telemetry labels only — NOT git selector labels
          region: us-east
          env: production

    # Enable backends by listing them (empty value = all defaults):
    network_discovery:
    device_discovery:
    snmp_discovery:
      ingest_buffer_size: 512               # backend-level tuning example
    gnmi_discovery:
    worker:
    snmp_telemetry:                         # Observability backends (metrics over OTLP, no Diode ingest)
      start_mode: on_demand                 # eager (default) | on_demand — start when the first policy arrives
      start_timeout: 60                     # seconds (1–300, default 30); only with on_demand
      policy_env_vars: [SNMP_COMMUNITY]
    gnmi_telemetry:
      start_mode: on_demand

  policies:
    # See backend-details.md for policy schemas per backend
    <backend_name>:
      <policy_name>:                        # forwarded verbatim; must not contain "/"
        config:
          schedule: "cron expression"
          timeout: 5                        # minutes (network_discovery); seconds (snmp_discovery)
          options: { ... }                  # backend-specific behaviour toggles
          defaults: { ... }                 # backend-specific NetBox defaults
        scope:
          # backend-specific targets
```

Every backend key accepts optional `host` / `port` overrides for its local API (defaults: device 8072, snmp 8070, network 8073, worker 8071, gnmi_discovery 8075, snmp_telemetry 8078, gnmi_telemetry 8079). `start_mode` / `start_timeout` work on any backend, not only the telemetry ones.

## Environment-Driven Overrides (`ORB_*`, v2.12+)

Any `orb.*` key can be set or overridden from the environment without templating the YAML. `__` separates path segments, single `_` stays inside a key, names are lower-cased:

```bash
ORB_SECRETS_MANAGER__ACTIVE=vault
ORB_SECRETS_MANAGER__SOURCES__VAULT__ADDRESS=http://127.0.0.1:8200
ORB_SECRETS_MANAGER__SOURCES__VAULT__AUTH=kubernetes
ORB_SECRETS_MANAGER__SOURCES__VAULT__AUTH_ARGS__ROLE=orb-agent
```

Rules: an empty value is ignored (never clobbers the file value); an unparseable name is skipped with a warning; a malformed value or two names mapping to one path fails startup. Overrides into `backends` or `policies` entries **replace the whole entry** (no deep merge) — keep those in YAML and use `ORB_*` for manager selection and scalar settings.

## Secret References

Environment variables and secrets-manager references can be used anywhere in the config (backend `${VAR}` support varies — see [backend-details.md](backend-details.md)):

```yaml
# Environment variable
password: ${MY_ENV_VAR}

# HashiCorp Vault — three grammars, in priority order:
password: ${vault://kv//network/credentials/admin_pass}   # qualified: mount // path / key (multi-segment mounts OK)
password: ${vault://network/credentials/admin_pass}       # short form: requires sources.vault.mount
password: ${vault://secret/data/network/credentials/admin_pass}   # legacy: single-segment mount

# Doppler (short or qualified)
password: ${doppler://CISCO_PASSWORD}
password: ${doppler://orb/prd/CISCO_PASSWORD}

# CyberArk CCP (beta)
password: ${cyberark://Lab-DB/cisco-svc-account}
username: ${cyberark://Lab-DB/cisco-svc-account/UserName}
password: ${cyberark://<AppID>//Lab-DB/cisco-svc-account}   # override the configured AppID

# Delinea Secret Server (beta)
password: ${delinea://id/42/password}
password: ${delinea://path/Servers/prod-db/password}

# Delinea DevOps Secrets Vault (DSV) — <secret-path>/<field-key>, split on the last "/"
password: ${dsv://servers/prod-db/password}
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
data only). Each policy file starts at the backend key (`device_discovery:`, not `orb.policies`).

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
  # Per-entity overrides (device_discovery, snmp_discovery, gnmi_discovery):
  device:
    description: "Discovered device"
  interface:
    description: "Discovered interface"
  ipaddress:            # device_discovery spelling
    description: "Discovered IP"
  # ip_address:         # snmp_discovery / gnmi_discovery spelling
  vlan:
    group: campus       # bare name (scoped to defaults.site) or {name, scope_site|scope_site_group|scope_region|scope_location}
```

## Interface Patterns

Device, SNMP and gNMI discovery support regex-based interface type matching. The keys are `match` and `type` (user patterns take precedence over built-in patterns and driver/SNMP-derived types):

```yaml
defaults:
  if_type: other                 # fallback when nothing matches
  interface_patterns:
    - match: "^(Ethernet|eth)"
      type: 1000base-t
    - match: "^(GigabitEthernet|ge-)"
      type: 1000base-t
    - match: "^(TenGigabitEthernet|xe-)"
      type: 10gbase-x-sfpp
    - match: "^(Loopback|lo)"
      type: virtual
    - match: "^(Vlan|vlan)"
      type: virtual
  interface_exclude_patterns:    # skip interfaces (and their IPs) entirely
    - "^veth"
    - "^docker"
```
