# Database Backup Strategy

## What Needs Backing Up

| What | Where | Why |
|------|-------|-----|
| PostgreSQL database | The DB itself | All content: users, journals, articles, reviews, settings |
| `src/files/` | On disk | Uploaded articles, PDFs, galleys, press files |
| `src/media/` | On disk | Profile images, summernote uploads, other media |

Your `settings.py` and any custom theme files should be in git (or at least copied somewhere safe), but the three items above are the critical data that lives outside of version control.

---

## Option 1: pg_dump (Recommended for Production)

This is the standard, reliable way to back up PostgreSQL. It creates a consistent snapshot even while the database is running.

### Manual backup

```bash
# Dump the database to a compressed SQL file
pg_dump -U janeway -h localhost -d janeway -Fc -f /backups/janeway_$(date +%Y%m%d_%H%M%S).dump

# Back up uploaded files alongside it
tar -czf /backups/files_$(date +%Y%m%d_%H%M%S).tar.gz /path/to/janeway/src/files/ /path/to/janeway/src/media/
```

### Restore from pg_dump

```bash
# Drop and recreate the database (or restore to a fresh one)
dropdb -U janeway janeway
createdb -U janeway janeway

# Restore
pg_restore -U janeway -h localhost -d janeway /backups/janeway_20260304_020000.dump

# Restore files
tar -xzf /backups/files_20260304_020000.tar.gz -C /
```

### Automate with cron

Create a backup script:

```bash
#!/bin/bash
# /opt/janeway-backup.sh

BACKUP_DIR="/backups/janeway"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
JANEWAY_DIR="/path/to/HeeroPublishing"
KEEP_DAYS=30

mkdir -p "$BACKUP_DIR"

# Database dump
pg_dump -U janeway -h localhost -d janeway -Fc \
  -f "$BACKUP_DIR/db_$TIMESTAMP.dump"

# Files backup
tar -czf "$BACKUP_DIR/files_$TIMESTAMP.tar.gz" \
  "$JANEWAY_DIR/src/files/" \
  "$JANEWAY_DIR/src/media/"

# Delete backups older than $KEEP_DAYS days
find "$BACKUP_DIR" -name "*.dump" -mtime +$KEEP_DAYS -delete
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$KEEP_DAYS -delete

echo "Backup completed: $TIMESTAMP"
```

Add to crontab:

```bash
chmod +x /opt/janeway-backup.sh

# Run daily at 2 AM
crontab -e
# Add this line:
0 2 * * * /opt/janeway-backup.sh >> /var/log/janeway-backup.log 2>&1
```

---

## Option 2: Janeway's Built-in Backup Command

Janeway has `python manage.py backup` which dumps the DB to JSON (`dumpdata`) and copies `files/` and `media/` into a zip. It works but is slower than pg_dump for large databases because it serializes everything to JSON.

### How to use it

```bash
# From within the project directory (or Docker container)
python src/manage.py backup
```

It creates a zip in `src/files/temp/` containing:
- `janeway.json` — full database dump (JSON)
- `files/` — all uploaded files
- `media/` — all media files

### Configure where backups go

In your `settings.py`:

```python
BACKUP_TYPE = "directory"          # "directory" or "s3"
BACKUP_DIR = "/backups/janeway/"   # where to copy the zip
BACKUP_EMAIL = True                # email admins on completion/failure
```

### Automate it

```bash
# Crontab entry — daily at 3 AM
0 3 * * * cd /path/to/HeeroPublishing && python src/manage.py backup >> /var/log/janeway-backup.log 2>&1
```

### Restore from JSON backup

```bash
# Unzip the backup
unzip 2026-03-04*.zip -d /tmp/restore/

# Load the database
python src/manage.py loaddata /tmp/restore/janeway.json

# Copy files back
cp -r /tmp/restore/files/* src/files/
cp -r /tmp/restore/media/* src/media/
```

> **Note:** `dumpdata`/`loaddata` can be fragile with complex foreign key relationships. pg_dump is more reliable for disaster recovery. Use the built-in command as a secondary/portable backup, not your only one.

---

## Option 3: Makefile Docker Backup (Development Only)

The repo includes `make db-save-backup` which tars the raw PostgreSQL data directory. This is a **filesystem-level copy** and should only be used for local dev snapshots — not production.

```bash
make db-save-backup                          # creates postgres-26-03-04-xxxxx.tar.gz
make db-load-backup BACKUP=postgres-26-03-04-xxxxx.tar.gz   # restores it
```

Don't rely on this for production. It requires stopping the database and doesn't handle file uploads.

---

## Recommended Strategy

| Layer | Method | Frequency | Retention |
|-------|--------|-----------|-----------|
| Primary | `pg_dump` + file tar | Daily (cron) | 30 days |
| Secondary | Copy backups off-server | Daily/weekly | 90 days |
| Portable | `manage.py backup` | Weekly (optional) | 4 copies |

### Off-server backup

Never keep all backups on the same server as the database. Sync to another location:

```bash
# rsync to another server
rsync -avz /backups/janeway/ backup-user@other-server:/backups/janeway/

# Or sync to an S3-compatible bucket (using aws cli or rclone)
rclone sync /backups/janeway/ remote:janeway-backups/
```

Add this as a second cron job that runs after the backup completes.

---

## Testing Your Backups

A backup you've never tested is not a backup. Periodically:

1. Spin up a test VPS or local Docker instance
2. Restore the pg_dump file to a fresh database
3. Restore the files tarball
4. Verify the site loads and content is intact
5. Check a few articles, user accounts, and uploaded files

Do this at least once after initial setup, then quarterly.
