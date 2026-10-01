# Backup & Restore Procedures

## Backup Checklist

| Component | Command / Location | Notes |
|-----------|-------------------|-------|
| Database | `pg_dump` | Core data |
| Media | `netbox/media/` | Uploaded images/files |
| Configuration | `configuration.py`, `ldap_config.py` | |
| Local packages | `local_requirements.txt` | |
| WSGI config | `gunicorn.py` | |
| Custom scripts | `SCRIPTS_ROOT` directory | |
| Secrets | `SECRET_KEY`, `API_TOKEN_PEPPERS` | If stored externally |

## Database Backup

```bash
# Full backup
pg_dump --username netbox --host localhost netbox > netbox_$(date +%F).sql

# Without changelog (saves significant space)
pg_dump --exclude-table-data=core_objectchange --username netbox netbox > netbox_slim.sql

# Schema only (dev reference)
pg_dump --username netbox --host localhost -s netbox > netbox_schema.sql
```

> **NetBox 4.7.0 dumps**: the ltree cascade triggers in a 4.7.0 `pg_dump` cannot be recreated on restore (fixed in 4.7.1). Upgrade to ≥4.7.1 before taking backups you intend to restore, and treat any existing 4.7.0 dump as needing the repair step below after restore.

## Media Backup

```bash
cd /opt/netbox
tar -czf netbox_media_$(date +%F).tar.gz netbox/media/
```

## Database Restore

```bash
# Drop and recreate (PostgreSQL user/permissions NOT included in dump)
psql -U postgres -c 'DROP DATABASE netbox'
psql -U postgres -c 'CREATE DATABASE netbox OWNER netbox'
psql -v ON_ERROR_STOP=1 -U netbox netbox < netbox_backup.sql && echo RESTORE_OK

# Custom-format (-Fc) dumps
pg_restore --exit-on-error -U netbox -d netbox netbox_backup.dump
```

> **Always restore with `ON_ERROR_STOP`** (or `pg_restore --exit-on-error`) and check the exit status. By default `psql` continues past errors and exits 0, so a restore missing an index, function, or trigger reports success. The known case is a **NetBox 4.7.0** dump: its ltree cascade triggers failed to recreate, so renames/moves of regions, site groups, locations, device roles, platforms, tenant/contact groups, wireless LAN groups, module bays, and inventory items stopped propagating to descendants. Expect dumps that used to "succeed" to now surface pre-existing errors (a role that already exists, an extension owned by another user) — fix those rather than dropping the flag.

After restore:
1. Verify PostgreSQL user `netbox` exists with correct permissions (4.7+: it needs `CREATE` on the database, or the `ltree` extension pre-installed)
2. Copy configuration files to correct locations
3. Extract media archive
4. Run `python manage.py migrate` (if restoring to a newer version)
5. *(4.7+)* Run `python manage.py rebuild_ltree_paths --check` (no locks). If it flags models, repair with `python manage.py rebuild_ltree_paths dcim.region dcim.location ...` — this rewrites every row of the named tables, so do it in a maintenance window. Always do this after restoring a 4.7.0 dump. See [Repairing Hierarchical Paths](https://netboxlabs.com/docs/netbox/administration/repairing-hierarchical-paths/)
6. Run `python manage.py reindex` to rebuild search index
7. Restart services: `sudo systemctl restart netbox netbox-rq`

## Automation Tips

- Schedule daily `pg_dump` via cron
- Retain N days of backups with rotation
- Test restores periodically — untested backups are not backups
- For large instances, consider PostgreSQL WAL archiving for point-in-time recovery
- `core_objectchange` is the fastest-growing table — excluding it from dev/test restores saves time
