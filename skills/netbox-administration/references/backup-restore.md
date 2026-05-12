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
psql -U netbox netbox < netbox_backup.sql
```

After restore:
1. Verify PostgreSQL user `netbox` exists with correct permissions
2. Copy configuration files to correct locations
3. Extract media archive
4. Run `python manage.py migrate` (if restoring to a newer version)
5. Run `python manage.py reindex` to rebuild search index
6. Restart services: `sudo systemctl restart netbox netbox-rq`

## Automation Tips

- Schedule daily `pg_dump` via cron
- Retain N days of backups with rotation
- Test restores periodically — untested backups are not backups
- For large instances, consider PostgreSQL WAL archiving for point-in-time recovery
- `core_objectchange` is the fastest-growing table — excluding it from dev/test restores saves time
