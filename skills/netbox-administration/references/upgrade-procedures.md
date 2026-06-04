# Upgrade Procedures

## Version Compatibility Matrix

| NetBox | Python | PostgreSQL | Redis | Django |
|--------|--------|-----------|-------|--------|
| 4.6 | 3.12–3.14 | 14+ (14 deprecated; 15+ required from v4.7) | 4.0+ | 6.0 |
| 4.5 | 3.12–3.14 | 14+ | 4.0+ | 5.x |
| 4.4 | 3.10–3.12 | 14+ | 4.0+ | 5.x |
| 4.3 | 3.10–3.12 | 14+ | 4.0+ | 5.x |
| 4.2 | 3.10–3.12 | 13+ | 4.0+ | 5.x |

> **4.6 platform shifts:** Django 6.0 (was 5.x) and **PostgreSQL 14 is deprecated** — v4.7 will require PostgreSQL 15+. Upgrade Postgres to 15+ before moving past 4.6. Also run **≥4.6.1** to avoid the template `environment_params` RCE (CVE-2026-29514) present in 4.6.0.

## Pre-Upgrade Checklist

1. **Read ALL release notes** between current and target version
2. **Back up** database and media (see [backup-restore.md](backup-restore.md))
3. **Check dependency versions** against the matrix above
4. **Major version jumps**: upgrade to latest minor of current major first (e.g., 3.x → 3.latest → 4.0)

## Tarball Upgrade

```bash
NEWVER=4.5.0
OLDVER=4.4.10

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
sudo git checkout v4.5.0
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

## Rollback

If upgrade fails:
1. Stop services
2. Restore database from backup
3. Point symlink back to old version: `sudo ln -sfn /opt/netbox-$OLDVER/ /opt/netbox`
4. Restart services
