# Upgrade Procedures

## Version Compatibility Matrix

| NetBox | Python | PostgreSQL | Redis | Django |
|--------|--------|-----------|-------|--------|
| 4.7 | 3.12–3.14 | **15+** plus the `ltree` extension | **6.0+** | 6.1 |
| 4.6 | 3.12–3.14 | 14+ (14 deprecated) | 5.0+ | 6.0 |
| 4.5 | 3.12–3.14 | 14+ | 5.0+ | 5.x |
| 4.4 | 3.10–3.12 | 14+ | 5.0+ | 5.x |
| 4.3 | 3.10–3.12 | 14+ | 5.0+ | 5.x |
| 4.2 | 3.10–3.12 | 13+ | 4.0+ | 5.x |

> **4.7 platform shifts:** PostgreSQL 15+ is now **required** — a `core` system check fails the `migrate` step on 14, where 4.6 only warned. Redis 5.x dropped. Django 6.1. Run **≥4.7.2**: 4.7.0/4.7.1 recorded the plaintext of tokens created via background bulk requests in job results, and 4.7.0 `pg_dump` output lost the ltree cascade triggers on restore. See [Upgrading to 4.7](#upgrading-to-47).
> **4.6 platform shifts:** Django 6.0 (was 5.x) and PostgreSQL 14 deprecated. Run **≥4.6.1** to avoid the template `environment_params` RCE (CVE-2026-29514) present in 4.6.0.

## Pre-Upgrade Checklist

1. **Read ALL release notes** between current and target version
2. **Back up** database and media (see [backup-restore.md](backup-restore.md))
3. **Check dependency versions** against the matrix above
4. **Major version jumps**: upgrade to latest minor of current major first (e.g., 3.x → 3.latest → 4.0)
5. **Plugins**: confirm each installed plugin supports the target release — e.g. netbox-branching needs v1.2.x on 4.7 (v1.1.x is the line for 4.4–4.6); see [netbox-branching](../../netbox-branching/SKILL.md)
6. **Queues**: let background queues drain (Admin > Background Tasks) before stopping workers — jobs enqueued on the old code may not run on the new (4.7: pre-upgrade `send_webhook` jobs fail with `TypeError`)

## Upgrading to 4.7

Two migrations scale with database size, hold locks, and are **not reversible in practice** (reversing restores the MPTT columns but does not repopulate them). Take a fresh backup and schedule a maintenance window.

### Before

```bash
# 1. PostgreSQL must be 15+ (the upgrade aborts on 14)
psql -U netbox -h localhost netbox -c 'SHOW server_version'

# 2. ltree extension: migrations CREATE it automatically if the NetBox DB user holds
#    CREATE on the database. Standard installs (netbox user owns the DB) already do.
sudo -u postgres psql -c 'GRANT CREATE ON DATABASE netbox TO netbox;'
# ...or have a DBA pre-install it:
sudo -u postgres psql netbox -c 'CREATE EXTENSION IF NOT EXISTS ltree;'

# 3. Redis must be 6.0+
redis-server -v
```

Then fix `configuration.py` (see [configuration-guide.md](configuration-guide.md)):

- Move `SENTRY_DSN` / `SENTRY_SAMPLE_RATE` / `SENTRY_SEND_DEFAULT_PII` / `SENTRY_TRACES_SAMPLE_RATE` into `SENTRY_CONFIG` — the old parameters are removed.
- If `RQ_DEFAULT_TIMEOUT` ≤ 60, set `WEBHOOK_DEFAULT_TIMEOUT` to a lower value or NetBox will not start.
- Rename `JINJA2_FILTERS` → `JINJA_FILTERS` (old name works until 5.0).
- Set `EMAIL['SERVER']` if NetBox sends mail (`ADMINS` error mail, logging handlers) — otherwise sends raise `InvalidMailer`.

Also: test SSO on a non-production 4.7 instance (`social-auth-app-django` 6.0 / `social-auth-core` 5.1 are major bumps), and drain the RQ queues.

### During

`upgrade.sh` runs the MPTT→ltree migration (schema changes under `ACCESS EXCLUSIVE`, which blocks reads and writes, then a per-table backfill that locks the rows it updates), followed by `rebuild_config_context_cache` (one `UPDATE` per device and VM). On large deployments expect minutes, not seconds. `rebuild_config_context_cache` skips already-populated objects, so it is safe to interrupt and re-run later; objects with an empty cache render config context on demand until it finishes.

### After

```bash
python netbox/manage.py rebuild_ltree_paths --check      # no locks; safe on a live system
python netbox/manage.py rebuild_config_context_cache     # only if the upgrade run was interrupted
```

Expect a brief lag before new or edited objects appear in global search — index updates are now a background job, so confirm `netbox-rq` is running.

## Tarball Upgrade

```bash
NEWVER=4.7.2
OLDVER=4.6.10

# Download and extract
wget https://github.com/netbox-community/netbox/archive/v$NEWVER.tar.gz
sudo tar -xzf v$NEWVER.tar.gz -C /opt
sudo ln -sfn /opt/netbox-$NEWVER/ /opt/netbox

# Copy config from old version
sudo cp /opt/netbox-$OLDVER/local_requirements.txt /opt/netbox/
sudo cp /opt/netbox-$OLDVER/netbox/netbox/configuration.py /opt/netbox/netbox/netbox/
sudo cp /opt/netbox-$OLDVER/netbox/netbox/ldap_config.py /opt/netbox/netbox/netbox/  # if exists
sudo cp -pr /opt/netbox-$OLDVER/netbox/media/ /opt/netbox/netbox/
sudo cp -r /opt/netbox-$OLDVER/netbox/scripts /opt/netbox/netbox/
sudo cp -r /opt/netbox-$OLDVER/netbox/reports /opt/netbox/netbox/
sudo cp /opt/netbox-$OLDVER/gunicorn.py /opt/netbox/
```

## Git Upgrade

```bash
cd /opt/netbox
sudo git fetch --tags
sudo git checkout v4.7.2
```

## Run Upgrade Script

```bash
sudo ./upgrade.sh

# If Python version mismatch:
sudo PYTHON=/usr/bin/python3.12 ./upgrade.sh

# For read-only DB standby node:
sudo ./upgrade.sh --readonly
```

The upgrade script handles:
- Rebuilding Python venv
- Installing requirements + `local_requirements.txt`
- Running database migrations
- Collecting static files
- Deleting stale content types
- Clearing expired sessions

## Post-Upgrade

```bash
sudo systemctl restart netbox netbox-rq
```

Verify:
- UI loads correctly
- API responds (`/api/status/`)
- Background jobs processing (`/admin/background-tasks/`)
- Search works (run `python manage.py reindex` if needed)
- *(4.7+)* Hierarchy paths consistent: `python manage.py rebuild_ltree_paths --check`

## Rollback

If upgrade fails:
1. Stop services
2. Restore database from backup
3. Point symlink back to old version: `sudo ln -sfn /opt/netbox-$OLDVER/ /opt/netbox`
4. Restart services

> **4.7**: the ltree migrations cannot be reversed with `migrate` — rolling back to 4.6 means restoring the pre-upgrade database backup.
