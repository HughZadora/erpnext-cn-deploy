# Backup and recovery guide

> Backups are the **lifeline** of a production environment. A system without
> backups can lose all its data at any moment.

---

## Why back up

ERPNext stores your customers, orders, and financial data. Once lost:

- Hard disk failure → everything gone
- Accidental database deletion → everything gone
- Upgrade problems → may need to roll back
- Attacked and encrypted → with a backup you don't need to pay ransom

**Not backing up = betting your data won't be lost.** That is not a worthwhile bet.

---

## What to back up

| What | How | Frequency |
|------|-----|-----------|
| Database | `bench --site all backup --with-files` | Daily (cron) |
| Site configuration | `~/frappe_docker/sites/` directory | On change |
| Docker volumes | `docker compose` volumes | On infrastructure changes |
| Environment file | `.env` copy | On changes |

---

## Backup method

### Manual backup (inside the container)

```bash
cd ~/erpnext/frappe_docker

# Database backup (bench backup creates a .sql.gz file)
docker compose exec backend bench --site erpnext.example.com backup --with-files
```

The backup file is stored in the container at:
```
/sites/<site-name>/private/backups/
```

### Copy out of the container

```bash
# Find the latest backup
docker compose exec backend ls -lht /home/frappe/frappe-bench/sites/erpnext.example.com/private/backups/ | head -5

# Copy to the host
docker compose cp backend:/home/frappe/frappe-bench/sites/erpnext.example.com/private/backups/<file>.sql.gz ~/erpnext/backups/
```

### Automated backup script (this repository)

This repository provides `scripts/backup.sh`, which:

1. Runs `bench backup --with-files` inside the container
2. Copies the backup files to the host
3. Compresses with timestamp
4. Retains the last N backups (configurable)
5. Optionally uploads to remote storage

### Scheduled backup (cron)

```bash
# Copy the backup script from this repository
cp ~/erpnext-deploy/scripts/backup.sh ~/erpnext/backup.sh

# Create the log directory
mkdir -p ~/erpnext/logs

# Test
bash ~/erpnext/backup.sh

# Configure daily backup (2:00 AM)
echo '0 2 * * * root /root/erpnext/backup.sh > /root/erpnext/logs/backup.log 2>&1' \
  | sudo tee /etc/cron.d/erpnext
```

---

## Restore procedure

### Restore database from a backup file

```bash
cd ~/erpnext/frappe_docker

# Copy the backup file into the container
docker compose cp ~/erpnext/backups/<backup>.sql.gz backend:/tmp/restore.sql.gz

# Restore the database
docker compose exec backend bench --site erpnext.example.com --force restore /tmp/restore.sql.gz

# Clear cache after restore
docker compose exec backend bench --site erpnext.example.com clear-cache
```

### Restore files (uploads and attachments)

```bash
# Copy the files backup into the container
docker compose cp ~/erpnext/backups/files-backup.tar.gz \
  backend:/home/frappe/frappe-bench/sites/erpnext.example.com/private/

# Extract
docker compose exec backend tar -xzf /home/frappe/frappe-bench/sites/erpnext.example.com/private/backups/files-backup.tar.gz \
  -C /home/frappe/frappe-bench/sites/erpnext.example.com/private/
```

### Full restore (fresh deployment)

If restoring to a new server:

1. Complete Steps 1-5 of the deployment guide (up to `docker compose up -d`)
2. Create the site with the same name (Step 6), using the same admin password
3. Skip installing ERPNext (Step 7) — the backup contains the app data
4. Restore the database backup (see above)
5. Restore the files backup (see above)
6. Run migrate and clear-cache
7. Run `bench --site erpnext.example.com doctor` to verify

---

## Backup verification

After each backup:

1. Check the file size (should be consistent with previous backups)
2. Test-restore to a temporary site
3. Verify the data is readable and complete

---

## Retention policy

| Type | Retention | Storage |
|------|-----------|---------|
| Daily backups | Keep the last 7 | Host local disk |
| Weekly backups | Keep the last 4 | Host local disk |
| Monthly backups | Keep the last 6 | Remote storage (if available) |

---

## Common issues

### Backup file is empty or too small

→ The site may not have enough data yet, or the database connection failed.
Check the container logs for errors.

### Backup takes too long

→ Large databases take time. For the first backup, run it manually and wait.
After that, the cron job handles it automatically.

### Restore fails with "table already exists"

→ The site already has data. Use `--force` or drop the existing tables first:
```bash
docker compose exec backend bench --site erpnext.example.com mariadb
# In the MariaDB shell: DROP DATABASE <site-db-name>; then recreate the site
```

---

## Checklist

- [ ] Backup script tested and working
- [ ] Scheduled backup configured (cron)
- [ ] Backup verified by test-restore
- [ ] Backup files stored outside the container
- [ ] Retention policy configured
