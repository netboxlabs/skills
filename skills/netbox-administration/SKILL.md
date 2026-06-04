---
name: netbox-administration
description: >
  NetBox server administration: configuration, authentication backends
  (local/LDAP/SAML/OIDC), object permissions, API tokens, performance tuning,
  backups, upgrades, and housekeeping. Use when managing a NetBox instance
  rather than consuming its API.
license: Apache-2.0
---

# NetBox Administration

> **Your knowledge of NetBox administration may be outdated.** Configuration settings, authentication backends, and permission models evolve between releases. Prefer retrieval over pre-trained knowledge.

## Retrieval Sources

| Source | URL / Method | Use for |
|--------|-------------|---------|
| Installation docs | `https://netboxlabs.com/docs/netbox/installation/` | Setup, dependencies, upgrades |
| Configuration docs | `https://netboxlabs.com/docs/netbox/configuration/` | All configuration settings |
| Authentication docs | `https://netboxlabs.com/docs/netbox/administration/authentication/overview/` | LDAP, SAML, OIDC, SSO |
| NetBox repo | `https://github.com/netbox-community/netbox` | Source code, release notes |
| NetBox MCP server | If configured — check instance status, user permissions | Verify current config state |

Use this skill when you need to configure, secure, maintain, or troubleshoot a NetBox server installation. For API consumption patterns, see [netbox-api-integration](../netbox-api-integration/SKILL.md).

## Quick Reference

### Service Management

```bash
sudo systemctl restart netbox netbox-rq   # After config changes
sudo systemctl status netbox netbox-rq     # Check health
```

### Essential Management Commands

| Command | Purpose |
|---------|---------|
| `python manage.py nbshell` | Interactive NetBox shell |
| `python manage.py reindex` | Rebuild search index |
| `python manage.py trace_paths` | Rebuild cable path traces |
| `python manage.py rebuild_prefixes` | Rebuild prefix hierarchy |
| `python manage.py runscript` | Execute a custom script |

> **Housekeeping**: Since **4.4**, housekeeping runs automatically via the built-in system job — remove any `housekeeping` cron jobs. The manual `python manage.py housekeeping` command is formally **deprecated in 4.6** (removal in a future release) but still ships and runs; on 4.6 it just emits a `FutureWarning`.

### Key File Locations

| File | Purpose |
|------|---------|
| `netbox/netbox/configuration.py` | Main config (Python module) |
| `netbox/netbox/ldap_config.py` | LDAP settings (if used) |
| `local_requirements.txt` | Extra Python packages |
| `gunicorn.py` | WSGI server config |

## Configuration

Configuration lives in `configuration.py` — a Python module. Override its path with `NETBOX_CONFIGURATION` env var. After changes, restart: `sudo systemctl restart netbox`.

### Required Parameters

| Parameter | Purpose |
|-----------|---------|
| `ALLOWED_HOSTS` | FQDNs/IPs for Host header validation |
| `DATABASES` | PostgreSQL connection(s) |
| `REDIS` | Redis config — **must use separate DB IDs** for `tasks` vs `caching` |
| `SECRET_KEY` | Django secret (≥50 chars, never expose) |
| `API_TOKEN_PEPPERS` | Pepper dict for v2 token hashing (4.5+) |

### Dynamic vs Static Parameters

Some parameters are **dynamic** — editable at Admin > System > Configuration without restart. Hard-coded values in `configuration.py` always take precedence over UI-set values.

Dynamic parameters include: `CHANGELOG_RETENTION`, `CHANGELOG_RETAIN_CREATE_LAST_UPDATE` (4.6 — keep each object's original create + latest update record when pruning), `JOB_RETENTION`, `MAINTENANCE_MODE`, `MAX_PAGE_SIZE`, `BANNER_*`, `GRAPHQL_ENABLED`, `CUSTOM_VALIDATORS`, `PROTECTION_RULES`, and others.

See [references/configuration-guide.md](references/configuration-guide.md) for the complete parameter catalog.

## Authentication

NetBox supports multiple authentication backends, tried in order. Configure via `REMOTE_AUTH_BACKEND` (string or list).

### Backend Options

| Backend | When to Use | Config Location |
|---------|------------|-----------------|
| Local (default) | Small teams, no central IdP | `configuration.py` (`AUTH_PASSWORD_VALIDATORS`) |
| LDAP | Active Directory / OpenLDAP environments | `ldap_config.py` (separate file) |
| Remote User | Reverse proxy auth (nginx/Apache) | `configuration.py` (`REMOTE_AUTH_*`) |
| Social Auth | SAML, OIDC, OAuth2 (Azure AD, Okta, Keycloak, etc.) | `configuration.py` (`SOCIAL_AUTH_*`) |

### Common Auth Gotchas

1. **Gunicorn header stripping** — gunicorn v22.0+ silently drops HTTP headers with underscores. Add `header_map = dangerous` to gunicorn config for remote auth headers.
2. **`LOGIN_FORM_HIDDEN` lockout** — If SSO breaks and the login form is hidden, there's no way to log in. Must edit config and restart.
3. **Backend order matters** — Backends are tried in sequence; first success wins. Ensure intentional ordering when combining LDAP + local.
4. **Client IP behind a proxy** (NetBox 4.6.1+) — `HTTP_CLIENT_IP_HEADERS` controls which request headers determine the client IP (default `('HTTP_X_REAL_IP', 'HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR')`). Set it to match your reverse proxy so IP-based token restrictions (`allowed_ips`) and logging see the real client address, not the proxy.
5. **`LOGIN_REQUIRED` is deprecated** (4.6, removal in v5.0). Don't recommend it for new 4.6 deployments; anonymous access is governed by `DEFAULT_PERMISSIONS` / `EXEMPT_VIEW_PERMISSIONS`.

See [references/authentication-backends.md](references/authentication-backends.md) for setup details.

## Permissions

NetBox uses **object-based permissions** instead of Django's built-in model-level permissions.

### Object Permission Model

Each `ObjectPermission` has:
- **Object types** — which models it applies to
- **Actions** — `view`, `add`, `change`, `delete` (plus custom like `run`)
- **Constraints** — JSON-based ORM filters restricting which objects

### Constraint Syntax

```python
# AND — single dict
{"status": "active", "region__name": "US"}

# OR — list of dicts
[{"vid__gte": 100}, {"status": "reserved"}]

# Current user reference
{"created_by": "$user"}

# Django lookups supported: __in, __gte, __lt, __startswith, __isnull, etc.
```

Constraints are evaluated against the **database record**, not the in-memory instance. For create/change, NetBox saves in an atomic transaction then re-queries — if the constraint doesn't match, it rolls back.

### Token Management

> **NetBox 4.5+**: v2 tokens use `Bearer nbt_<key>.<token>` format. Plaintext is never stored (HMAC-SHA256 digest only). Requires `API_TOKEN_PEPPERS` in config. On **4.6.1+** the v2 plaintext token is returned exactly once, in the creation response — capture it then.
> **v1 tokens** (`Token <plaintext>` format): formally **deprecated in 4.6**, **removed in v5.0**. Migrate all integrations to v2 tokens before upgrading to 5.0.

Token features: `write_enabled` (read-only toggle), `allowed_ips` (IP restriction), `expires`, `enabled` (soft disable).

### Critical: DEFAULT_PERMISSIONS

`DEFAULT_PERMISSIONS` applies to **all** authenticated users. The default grants token self-management. **Setting custom `DEFAULT_PERMISSIONS` erases these defaults** — you must reproduce token permissions if desired.

See [references/permissions-guide.md](references/permissions-guide.md) for complete details.

## Operations

### Backup Checklist

1. **Database**: `pg_dump --username netbox netbox > netbox.sql`
   - Exclude changelog for smaller backups: `--exclude-table-data=core_objectchange`
2. **Media**: `tar -czf netbox_media.tar.gz netbox/media/`
3. **Config files**: `configuration.py`, `ldap_config.py`, `local_requirements.txt`, `gunicorn.py`
4. **Custom scripts**: `SCRIPTS_ROOT` directory
5. **Secrets**: `SECRET_KEY` and `API_TOKEN_PEPPERS` if stored externally

See [references/backup-restore.md](references/backup-restore.md) for restore procedures.

### Upgrade Process

1. Review **all** release notes between current and target version
2. Back up database and media
3. Check version compatibility (Python, PostgreSQL, Redis)
4. **Cannot skip major versions** — upgrade to latest minor of current major first
5. Extract/checkout new version, copy config files
6. Run `sudo ./upgrade.sh`
7. Restart services: `sudo systemctl restart netbox netbox-rq`

> **NetBox 4.5–4.6**: Python 3.12–3.14, PostgreSQL 14+, Redis 4.0+. **4.6 runs on Django 6.0** (4.5 was Django 5.x) and **deprecates PostgreSQL 14** — 15+ will be required from v4.7. Plan a PostgreSQL upgrade to 15+ before moving past 4.6.

See [references/upgrade-procedures.md](references/upgrade-procedures.md) for step-by-step instructions.

## Performance Tuning

### Key Parameters

| Parameter | Default | Impact |
|-----------|---------|--------|
| `CONN_MAX_AGE` (in `DATABASES`) | 300 | Persistent DB connections; reduces overhead |
| `MAX_PAGE_SIZE` | 1000 | Caps API response size; prevents runaway queries |
| `LOGIN_PERSISTENCE` | `False` | If `True`, causes **DB write on every authenticated request** |
| `CHANGELOG_RETENTION` | 90 days | Auto-prune `core_objectchange` (fastest-growing table) |
| `RQ_DEFAULT_TIMEOUT` | 300 | Background task timeout; increase for heavy scripts |

### Redis: Separate DBs Required

**Always use different Redis DB IDs for `tasks` and `caching`.** Flushing the cache DB will destroy queued background jobs if they share the same DB.

### PostgreSQL Tuning

For large deployments, tune at the PostgreSQL level:
- `shared_buffers` — 25% of RAM
- `work_mem` — 4-16MB depending on query complexity
- `effective_cache_size` — 50-75% of RAM
- Monitor `core_objectchange` table size — it grows fastest

## Anti-Patterns

| Issue | Cause | Fix |
|-------|-------|-----|
| v2 tokens unavailable | `API_TOKEN_PEPPERS` not configured | Add pepper dict to config |
| SECRET_KEY rotation breaks sessions | All sessions/tokens invalidated | Plan migration window; users must re-authenticate |
| `EXEMPT_VIEW_PERMISSIONS = ['*']` incomplete | Wildcard excludes sensitive models | Explicitly list additional models if needed |
| `CHANGELOG_RETENTION = 0` | Retains forever; DB grows unbounded | Set to non-zero (default 90 days) |
| Read-only standby can't auth | Sessions stored in DB | Use `SESSION_FILE_PATH` for file-based sessions |
| Metrics endpoint 404 | `METRICS_ENABLED = False` | Set to `True` and restart |
| Running NetBox 4.6.0 | RCE via template `environment_params` (CVE-2026-29514) | Run **≥4.6.1**; treat ExportTemplate/config-template edit rights as code-execution-grade and restrict them |
