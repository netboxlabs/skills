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
| `python manage.py runscript` | Execute a custom script (core scripts **deprecated in 4.7**, removed in 5.0 — see [netbox-custom-scripts](../netbox-custom-scripts/SKILL.md)) |
| `python manage.py rebuild_config_context_cache [--force]` | *(4.7)* Populate the pre-rendered config context cache; `--force` re-renders populated entries after writes that bypassed signals |
| `python manage.py rebuild_ltree_paths [--check] [app.Model ...]` | *(4.7)* Detect (`--check`, no locks) or repair stale hierarchy `path`/`sort_path` columns |

> **Housekeeping**: Since **4.4**, housekeeping runs automatically via the built-in system job — remove any `housekeeping` cron jobs. The manual `python manage.py housekeeping` command is **removed in 4.7**; on 4.6 it still runs but emits a `FutureWarning`.

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

> **NetBox 4.7 configuration changes** — fix these in `configuration.py` before upgrading:
> - `SENTRY_DSN`, `SENTRY_SAMPLE_RATE`, `SENTRY_SEND_DEFAULT_PII`, `SENTRY_TRACES_SAMPLE_RATE` are **removed** — set them as keys of `SENTRY_CONFIG` (`dsn`, `sample_rate`, ...). `SENTRY_ENABLED = True` with no `dsn` in `SENTRY_CONFIG` refuses to start.
> - `JINJA2_FILTERS` renamed `JINJA_FILTERS` (old name works with a `DeprecationWarning` until 5.0).
> - `EMAIL['SERVER']` is **mandatory** to send mail — if omitted, sends raise `InvalidMailer` at send time, not at startup.
> - New `WEBHOOK_DEFAULT_TIMEOUT` (default 60 s) **must be lower than `RQ_DEFAULT_TIMEOUT`** or NetBox refuses to start — if you lowered `RQ_DEFAULT_TIMEOUT` to ≤ 60, set it explicitly.
> - New `BULK_UPDATE_CHUNK_SIZE` (default 5000) bounds rows per bulk `UPDATE`; custom-field create-with-default/delete moves to a background job above this many objects.

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
5. **`LOGIN_REQUIRED` is deprecated** (4.6, removal in v5.0). Don't recommend it for new deployments; anonymous access is governed by `DEFAULT_PERMISSIONS` / `EXEMPT_VIEW_PERMISSIONS`.
6. **SSO library major bumps in 4.7** — `social-auth-app-django` 6.0 / `social-auth-core` 5.1. Test every SAML/OIDC backend on a non-production 4.7 instance before upgrading; keep a local-credential admin (and `LOGIN_FORM_HIDDEN = False`) until verified.

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

> **NetBox 4.7+**: the `token` field is **read-only on REST create** — clients can no longer choose the plaintext (any supplied value is ignored). Executing a custom script via REST requires a token with `write_enabled = true`. **4.7.0/4.7.1** recorded the plaintext of tokens created via `?background=true` bulk requests in job results (readable by anyone who can view jobs) — run **≥4.7.2** and replace any token created that way.

Token features: `write_enabled` (read-only toggle; *(4.7)* also gates REST script execution), `allowed_ips` (IP restriction), `expires`, `enabled` (soft disable).

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
6. **Restore** with `psql -v ON_ERROR_STOP=1` (or `pg_restore --exit-on-error`) and check the exit status — plain `psql` reports success on a partial restore. A **4.7.0** dump restored that way silently lost the ltree cascade triggers (fixed 4.7.1); after restoring or upgrading from 4.7.0, run `python manage.py rebuild_ltree_paths --check`.

See [references/backup-restore.md](references/backup-restore.md) for restore procedures.

### Upgrade Process

1. Review **all** release notes between current and target version; confirm every plugin supports the target release
2. Back up database and media
3. Check version compatibility (Python, PostgreSQL, Redis) against the matrix in the reference
4. **Cannot skip major versions** — upgrade to latest minor of current major first
5. Let the RQ queues drain (Admin > Background Tasks) before stopping workers
6. Extract/checkout new version, copy config files
7. Run `sudo ./upgrade.sh`
8. Restart services: `sudo systemctl restart netbox netbox-rq`

> **NetBox 4.7**: Python 3.12–3.14, **PostgreSQL 15+ required** (a system check aborts `migrate` on 14), **Redis 6.0+ required**, Django 6.1. The PostgreSQL **`ltree` extension** is required; migrations install it automatically if the NetBox DB user has `CREATE` on the database (`GRANT CREATE ON DATABASE netbox TO netbox;`), or a DBA can pre-install it with `CREATE EXTENSION IF NOT EXISTS ltree;`. The upgrade is **long and effectively irreversible**: the MPTT→ltree migration holds `ACCESS EXCLUSIVE` locks while it backfills, then `rebuild_config_context_cache` issues one `UPDATE` per device/VM — take a backup and plan a maintenance window. Queued `send_webhook` jobs from before the upgrade fail with `TypeError`, so drain queues first. Run **≥4.7.2** (4.7.0/4.7.1 token-plaintext exposure; 4.7.0 `pg_dump` trigger loss).
> **NetBox 4.5–4.6**: Python 3.12–3.14, PostgreSQL 14+ (14 **deprecated** in 4.6 — move to 15+ before 4.7), Redis 5.0+. 4.6 runs Django 6.0, 4.5 Django 5.x.

See [references/upgrade-procedures.md](references/upgrade-procedures.md) for step-by-step instructions.

## Performance Tuning

### Key Parameters

| Parameter | Default | Impact |
|-----------|---------|--------|
| `CONN_MAX_AGE` (in `DATABASES`) | 300 | Persistent DB connections; reduces overhead |
| `MAX_PAGE_SIZE` | 1000 | Caps API response size; prevents runaway queries |
| `LOGIN_PERSISTENCE` | `False` | If `True`, causes **DB write on every authenticated request** |
| `CHANGELOG_RETENTION` | 90 days | Auto-prune `core_objectchange` (fastest-growing table) |
| `RQ_DEFAULT_TIMEOUT` | 300 | Background task timeout; increase for heavy scripts. *(4.7)* Must exceed `WEBHOOK_DEFAULT_TIMEOUT` |
| `WEBHOOK_DEFAULT_TIMEOUT` *(4.7)* | 60 | Per-request webhook timeout (1–3600 s); a webhook's own `timeout` field overrides it |
| `BULK_UPDATE_CHUNK_SIZE` *(4.7)* | 5000 | Rows per bulk `UPDATE` statement; lower it if large backfills hit `statement_timeout` (`None` disables chunking) |

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
| Running NetBox 4.7.0/4.7.1 | Plaintext of tokens created via `?background=true` bulk requests recorded in job results | Run **≥4.7.2**; treat such tokens as exposed and replace them |
| NetBox won't start after 4.7 upgrade (webhook timeout error) | `RQ_DEFAULT_TIMEOUT` ≤ 60 with the default `WEBHOOK_DEFAULT_TIMEOUT = 60` | Set `WEBHOOK_DEFAULT_TIMEOUT` below `RQ_DEFAULT_TIMEOUT` |
| `InvalidMailer` when NetBox sends mail (4.7) | `EMAIL['SERVER']` not set | Set `EMAIL = {'SERVER': ..., ...}` — Django `EMAIL_*` settings are no longer populated |
| New/edited objects missing from global search (4.7) | Search index updates deferred to a background job | Ensure `netbox-rq` is running; a brief lag is expected (updates are synchronous only when no worker exists) |
| Renamed/moved region, location, etc. not reflected in descendants (4.7) | Database restored from a 4.7.0 dump without `ON_ERROR_STOP` — ltree triggers missing | Upgrade to ≥4.7.1, `rebuild_ltree_paths --check`, then `rebuild_ltree_paths <app.model>` |

## Version Notes

### NetBox 4.7 (2026-09-02)

- **Platform**: PostgreSQL 15+ with `ltree`, Redis 6.0+, Django 6.1. Long, effectively irreversible upgrade — see [references/upgrade-procedures.md](references/upgrade-procedures.md). Run **≥4.7.2**.
- **Removed**: `housekeeping` management command; `SENTRY_DSN`/`SENTRY_SAMPLE_RATE`/`SENTRY_SEND_DEFAULT_PII`/`SENTRY_TRACES_SAMPLE_RATE` (use `SENTRY_CONFIG`).
- **Config**: `JINJA2_FILTERS` → `JINJA_FILTERS`; new `WEBHOOK_DEFAULT_TIMEOUT`, `BULK_UPDATE_CHUNK_SIZE`; `EMAIL['SERVER']` mandatory; Django `MAILERS` replaces the `EMAIL_*` settings (plugin code reading `EMAIL_HOST` etc. or calling `get_connection()` with an explicit backend breaks).
- **Security**: REST token `token` read-only on create; REST script execution needs a write-enabled token; custom link `request` context is a sanitized subset (`id`, `path`, `path_info`, `method`, `GET`, `user` — no cookies, headers, or session); URL custom fields validated against `ALLOWED_URL_SCHEMES` (scheme-less values stored as `https://`).
- **Behavior**: global search index updates deferred to a background job; custom field create-with-default/delete deferred to a job above `BULK_UPDATE_CHUNK_SIZE` objects; config context pre-rendered and cached per device/VM (`rebuild_config_context_cache`).
- **Deprecated**: core custom scripts (supported through 4.8, removed 5.0) and `SCRIPTS_ROOT` with them — see [netbox-custom-scripts](../netbox-custom-scripts/SKILL.md).
- **Install path**: NetBox is published to PyPI; the Python-package install/upgrade (`netbox upgrade --no-input`) is **experimental** — not for production.

### NetBox 4.6 (2026-05-05)

- Django 6.0; PostgreSQL 14 deprecated; `housekeeping` command deprecated (`FutureWarning`); `LOGIN_REQUIRED` deprecated; v1 tokens deprecated (removed 5.0); v2 plaintext returned once at creation and `HTTP_CLIENT_IP_HEADERS` added (4.6.1+); `CHANGELOG_RETAIN_CREATE_LAST_UPDATE` added. Run **≥4.6.1** (CVE-2026-29514).
