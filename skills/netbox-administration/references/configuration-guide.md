# Configuration Parameter Reference

## Required Parameters

| Parameter | Purpose | Notes |
|-----------|---------|-------|
| `ALLOWED_HOSTS` | Host header validation | List of FQDNs/IPs; also sets `CSRF_TRUSTED_ORIGINS` |
| `DATABASES` | PostgreSQL connection(s) | Engine must be PostgreSQL-compatible. `CONN_MAX_AGE=300` default |
| `REDIS` | Redis for tasks + caching | **Separate DB IDs required** for `tasks` vs `caching` |
| `SECRET_KEY` | Django secret key | ≥50 chars. Changing invalidates all sessions/tokens |
| `API_TOKEN_PEPPERS` | v2 token peppers (4.5+) | Dict `{id: pepper}`, each ≥50 chars. Start with ID 1 |

## Security & Authentication Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `AUTH_PASSWORD_VALIDATORS` | 12-char min + alphanumeric | Password policy for local accounts |
| `CORS_ORIGIN_ALLOW_ALL` | `False` | Allow all CORS origins |
| `CORS_ORIGIN_WHITELIST` | `[]` | Allowed CORS origins list |
| `CSRF_COOKIE_SECURE` | `False` | HTTPS-only CSRF cookie |
| `DEFAULT_PERMISSIONS` | Token self-management | Auto-applied to all authenticated users |
| `EXEMPT_VIEW_PERMISSIONS` | `[]` | Models viewable without auth (`['*']` for most) |
| `LOGIN_FORM_HIDDEN` | `False` | Hide login form (SSO-only deployments) |
| `LOGIN_PERSISTENCE` | `False` | Reset session TTL on each request (causes DB writes) |
| `LOGIN_REQUIRED` | `True` (since 4.0.2) | Require auth for all access. **Deprecated in 4.6, removed in v5.0** — govern anonymous access via `DEFAULT_PERMISSIONS` / `EXEMPT_VIEW_PERMISSIONS` instead |
| `LOGIN_TIMEOUT` | 1209600 (14 days) | Session cookie lifetime in seconds |
| `SECURE_HSTS_SECONDS` | 0 | HSTS header duration |
| `SECURE_SSL_REDIRECT` | `False` | Force HTTPS redirect |
| `SESSION_COOKIE_SECURE` | `False` | HTTPS-only session cookie |
| `SESSION_FILE_PATH` | `None` | File-based sessions (for read-only DB standby) |

## Remote Authentication Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `REMOTE_AUTH_ENABLED` | `False` | Enable remote auth |
| `REMOTE_AUTH_BACKEND` | `'netbox.authentication.RemoteUserBackend'` | Backend class(es) |
| `REMOTE_AUTH_HEADER` | `'HTTP_REMOTE_USER'` | Header for username |
| `REMOTE_AUTH_AUTO_CREATE_USER` | `False` | Auto-create local accounts |
| `REMOTE_AUTH_AUTO_CREATE_GROUPS` | `False` | Auto-create groups |
| `REMOTE_AUTH_DEFAULT_GROUPS` | `[]` | Groups for new remote users |
| `REMOTE_AUTH_DEFAULT_PERMISSIONS` | `{}` | Permissions for new remote users |
| `REMOTE_AUTH_GROUP_SYNC_ENABLED` | `False` | Sync groups on each login |
| `REMOTE_AUTH_GROUP_HEADER` | `'HTTP_REMOTE_USER_GROUP'` | Header for group names |
| `REMOTE_AUTH_GROUP_SEPARATOR` | `'\|'` | Separator in group header |
| `REMOTE_AUTH_SUPERUSER_GROUPS` | `[]` | Groups granting superuser |
| `REMOTE_AUTH_SUPERUSERS` | `[]` | Usernames with superuser |

## System / Operational Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `BASE_PATH` | `''` | Sub-path behind reverse proxy |
| `DEBUG` | `False` | **Never in production** |
| `DEFAULT_LANGUAGE` | `'en-us'` | UI language |
| `EMAIL` | localhost:25 | Email server (SERVER, PORT, USERNAME, PASSWORD, USE_SSL, USE_TLS, FROM_EMAIL) |
| `HOSTNAME` | system hostname | Display name in UI (4.4+) |
| `HTTP_CLIENT_IP_HEADERS` | `('HTTP_X_REAL_IP', 'HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR')` | Request headers checked (in order) to determine client IP (4.6.1+). Set to match your reverse proxy so `allowed_ips` token restrictions and logging see the real client address |
| `HTTP_PROXIES` | `None` | Outbound proxy config |
| `ISOLATED_DEPLOYMENT` | `False` | Disables internet-dependent features |
| `LOGGING` | `{}` | Django logging config |
| `MEDIA_ROOT` | `$INSTALL/netbox/media` | Uploaded files location |
| `METRICS_ENABLED` | `False` | Prometheus `/metrics` endpoint |
| `PLUGINS` / `PLUGINS_CONFIG` | `[]` / `{}` | Plugin configuration |
| `RQ_DEFAULT_TIMEOUT` | 300 | Background task timeout (seconds) |
| `RQ_RETRY_MAX` | 0 | Max retries for failed jobs |
| `SCRIPTS_ROOT` | `$INSTALL/netbox/scripts/` | Custom script file path |
| `SEARCH_BACKEND` | `CachedValueSearchBackend` | Search engine backend |
| `STORAGES` | Local filesystem | Django storages config (supports S3) |
| `TIME_ZONE` | `'UTC'` | Server timezone |

## Available Loggers

For `LOGGING` config: `netbox.auth.*`, `netbox.api.views.*`, `netbox.event_rules`, `netbox.jobs.*`, `netbox.scripts.*`, `netbox.views.*`

## Dynamic Parameters (UI-Editable)

These can be changed at Admin > System > Configuration without restart. Hard-coded values in `configuration.py` take precedence.

`BANNER_*`, `CHANGELOG_RETENTION` (default 90 days), `CHANGELOG_RETAIN_CREATE_LAST_UPDATE` (4.6 — when pruning per `CHANGELOG_RETENTION`, retain each object's original create + most-recent-update change record), `JOB_RETENTION` (default 90 days), `MAINTENANCE_MODE`, `MAX_PAGE_SIZE` (default 1000), `PAGINATE_COUNT`, `GRAPHQL_ENABLED`, `CUSTOM_VALIDATORS`, `PROTECTION_RULES`, `ENFORCE_GLOBAL_UNIQUE`, `ALLOWED_URL_SCHEMES`, `DEFAULT_USER_PREFERENCES`, `MAPS_URL`, `PREFER_IPV4`
