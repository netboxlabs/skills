# Deployment Patterns Reference

## Docker — Standard Deployment

```bash
docker run --net=host \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID=your-client-id \
  -e DIODE_CLIENT_SECRET=your-client-secret \
  netboxlabs/orb-agent:2.15.0 run -c /opt/orb/agent.yaml
```

- `--net=host` is required for NMAP SYN scans (network_discovery)
- Alternative: `-u root` instead of `--net=host`
- Mount the directory containing `agent.yaml` to `/opt/orb/`
- Pin the image tag (Docker Hub tags are bare semver: `2.15.0`); `latest` tracks the newest release

### Running as a service

```bash
docker run -d --name orb-agent --restart unless-stopped \
  --stop-timeout 60 \
  --log-driver local \
  --net=host \
  -v /local/orb:/opt/orb/ \
  --env-file /local/orb/.env \
  netboxlabs/orb-agent:2.15.0 run -c /opt/orb/agent.yaml
```

- `--stop-timeout 60`: the agent stops backends one at a time and finalizes in-flight policy runs, which can exceed Docker's 10 s default
- `--log-driver local`: the default `json-file` driver never rotates; `local` keeps 5 × 20 MB compressed files
- `systemctl enable --now docker` (or `podman-restart.service`, plus `loginctl enable-linger` for rootless) so the restart policy applies at boot
- `docker restart orb-agent` applies an `agent.yaml` change; to upgrade, `docker stop` then `rm` (never `rm -f`, which sends SIGKILL)
- Compose equivalent: `restart: unless-stopped` + `stop_grace_period: 60s`

## Podman

### Privileged (full functionality)

```bash
sudo podman run --privileged --net=host \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:2.15.0 run -c /opt/orb/agent.yaml
```

### Rootless (restricted)

```bash
podman run \
  -v ${PWD}:/opt/orb/ \
  -e DIODE_CLIENT_ID -e DIODE_CLIENT_SECRET \
  netboxlabs/orb-agent:2.15.0 run -c /opt/orb/agent.yaml
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
        mount: "kv"              # Optional default KV-v2 mount → enables the short form below
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
            username: ${vault://kv//network/switch_user}   # qualified: mount // path / key
            password: ${vault://network/switch_pass}       # short form (uses sources.vault.mount)
```

Use the qualified `//` form for multi-segment mounts (`${vault://secret/team-a//network/switch_pass}`); the legacy single-slash form `${vault://kv/network/switch_pass}` still parses for single-segment mounts.

### Supported Vault Auth Methods

| Method | Key Fields |
|--------|-----------|
| `token` | `token` |
| `approle` | `role_id`, `secret_id` |
| `userpass` | `username`, `password` |
| `kubernetes` | `role`, (uses in-cluster service account) |
| `ldap` | `username`, `password` |

### Selecting the secrets manager from the environment (v2.12+)

Keep `agent.yaml` generic and inject the secrets-manager choice per deployment with `ORB_*` overrides — no templating needed:

```yaml
# Kubernetes pod spec (excerpt): Vault via the pod's ServiceAccount, no static token
env:
  - name: ORB_SECRETS_MANAGER__ACTIVE
    value: "vault"
  - name: ORB_SECRETS_MANAGER__SOURCES__VAULT__ADDRESS
    value: "http://vault:8200"
  - name: ORB_SECRETS_MANAGER__SOURCES__VAULT__AUTH
    value: "kubernetes"
  - name: ORB_SECRETS_MANAGER__SOURCES__VAULT__AUTH_ARGS__ROLE
    value: "orb-agent"
```

Other providers: `active: doppler | cyberark | delinea | dsv` — see [agent-config-format.md](agent-config-format.md#secret-references) for each provider's source keys and reference syntax.

## Git-based Fleet Management

For managing multiple agents from a central policy repo:

### Agent Configuration

```yaml
orb:
  labels:                 # top-level — what the git selector matches against
    region: us-east
    site: dc1
    env: production

  config_manager:
    active: git
    sources:
      git:
        url: https://github.com/org/orb-policies.git   # NOT "repo:"
        branch: main
        schedule: "*/5 * * * *"   # poll for policy changes; omit = fetch once at startup
        auth: basic       # string, NOT auth.type — basic | ssh | github_app
        username: ${GIT_USER}
        password: ${GIT_TOKEN}
        skip_tls: false

  backends:
    common:
      diode:
        target: grpc://diode-server:8080/diode  # Cloud/Enterprise: use https://
        client_id: ${DIODE_CLIENT_ID}
        client_secret: ${DIODE_CLIENT_SECRET}
        agent_name: agent-us-east-01
      agent_labels:       # telemetry labels on exported data — NOT selector matching
        managed_by: orb

    network_discovery:
    device_discovery:
    snmp_discovery:
```

### Policy Repo — selector.yaml

A **map of named selector blocks**. Each block's `selector:` is matched against
the agent's top-level `orb.labels`; `policies:` maps a policy name to its file path:

```yaml
agent_us_east:
  selector:
    region: us-east
  policies:
    network_policy:
      path: policies/us-east-network.yaml
    device_policy:
      path: policies/us-east-devices.yaml

agent_eu_west:
  selector:
    region: eu-west
  policies:
    network_policy:
      path: policies/eu-west-network.yaml

agent_all_prod:
  selector:
    env: production
  policies:
    snmp_policy:
      path: policies/production-snmp.yaml
      enabled: true       # optional; false to skip
```

Each referenced policy file starts at the backend key (`device_discovery:` …) — not at `orb.policies`. `orb.backends`, `orb.config_manager`, `orb.labels` and the Diode credentials stay in the local `agent.yaml`; only policies and `selector.yaml` live in Git.

### GitHub App authentication (v2.15+, github.com only)

Short-lived, repo-scoped installation tokens instead of a personal access token. Create a GitHub App with **Contents: Read**, install it on only the policy repo, generate a private key, then:

```yaml
config_manager:
  active: git
  sources:
    git:
      url: https://github.com/org/orb-policies.git
      schedule: "*/5 * * * *"
      auth: github_app
      github_app:
        client_id: "Iv23liAbCdEfGhIjKlMn"      # the app's Client ID (numeric App ID also works)
        installation_id: "78901234"            # from the installation settings URL — NOT the App ID
        private_key: /opt/orb/github-app.pem   # or the PEM content, e.g. ${GITHUB_APP_KEY}
```

Mount the `.pem` read-only (`-v /local/orb/github-app.pem:/opt/orb/github-app.pem:ro`). The agent mints a one-hour token at startup (so a wrong id or key fails fast) and re-mints it automatically. Not supported for GitHub Enterprise Server or `*.ghe.com` — use `auth: basic` with a PAT there. For `auth: ssh`, mount a `known_hosts` file and set `SSH_KNOWN_HOSTS=/opt/orb/known_hosts`.

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
