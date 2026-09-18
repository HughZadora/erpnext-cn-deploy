# Operations manual

> Daily operations: this one document is enough. Covers common commands,
> troubleshooting, and upgrade steps.

---

## Quick reference card

### The 10 most-used commands

```bash
# Working directory
cd ~/erpnext/frappe_docker

docker compose ps                              # 1. Check all service states
docker compose logs -f --tail=100 backend      # 2. View backend logs
docker compose up -d                           # 3. Start
docker compose stop                            # 4. Pause
docker compose restart backend                 # 5. Restart a specific service
docker compose exec backend bench doctor       # 6. Health check
docker compose exec backend bench migrate      # 7. Database migration
bash ~/erpnext/backup.sh                       # 8. Manual backup
bash ~/erpnext/monitor.sh                      # 9. System overview
df -h                                          # 10. Check disk space
```

### Container state interpretation

```bash
docker compose ps
# All containers should show "Up" status:
#   Up          → running normally
#   Up (healthy) → running + health check passing
#   Exit 0      → stopped cleanly (check if intentional)
#   Exit 1      → crashed (check logs)
#   Restarting  → crashing and being restarted by Docker (check logs)
```

---

## Daily operations

### Check system status

```bash
docker compose ps        # All containers should be Up
docker compose logs --tail=20 backend  # Look for errors in recent logs
df -h                    # Disk usage; /var/lib/docker is the biggest consumer
free -h                  # Memory usage
```

### View service logs

```bash
# Backend (the most important container)
docker compose logs -f --tail=100 backend

# Database
docker compose logs -f --tail=100 db

# Redis (cache and queue)
docker compose logs -f --tail=50 redis-cache
docker compose logs -f --tail=50 redis-queue

# Frontend (nginx proxy)
docker compose logs -f --tail=50 frontend
```

### Restart a service

```bash
# Restart a specific container
docker compose restart backend
docker compose restart frontend
docker compose restart db

# Restart everything
docker compose restart
```

---

## Routine maintenance

### Weekly checks

```bash
# Disk space
df -h

# Container resource usage
docker stats --no-stream

# Database size
docker compose exec db mysql -u root -p"$(grep DB_PASSWORD .env | cut -d= -f2)" \
  -e "SELECT table_schema AS 'db', ROUND(SUM(data_length+index_length)/1024/1024,1) AS 'size_mb' FROM information_schema.tables GROUP BY table_schema ORDER BY size_mb DESC;"

# Tailscale connectivity (if using Tailscale)
tailscale status
```

### Monthly maintenance

```bash
# Update system packages
apt update && apt upgrade -y

# Update Docker
apt install --only-upgrade docker-ce docker-ce-cli containerd.io

# Prune old Docker images
docker image prune -a --filter "until=168h"

# Check Docker disk usage
docker system df
```

---

## Upgrade ERPNext

### Standard upgrade

```bash
cd ~/erpnext/frappe_docker

# Pull the latest images
docker compose pull

# Migrate the database
docker compose exec backend bench --site erpnext.example.com migrate

# Restart all containers
docker compose up -d

# Verify
docker compose exec backend bench --site erpnext.example.com doctor
docker compose ps
```

### Version pinning

For production stability, pin to a specific ERPNext version in `.env`:

```env
ERPNEXT_VERSION=v15.x.x
FRAPPE_VERSION=v15.x.x
```

### Rollback

If an upgrade breaks:

```bash
# Roll back to the previous image
docker compose down
# Edit .env to the previous version tags
docker compose pull
docker compose up -d
```

---

## Troubleshooting

### Container keeps restarting

```bash
# Check the logs
docker compose logs --tail=100 <container-name>

# Common causes:
# - Out of memory (check docker stats)
# - Configuration error (check .env)
# - Database connection failure (check db container)
```

### Website is slow

```bash
# Check resource usage
docker stats --no-stream

# Check for large log files
docker compose logs --tail=1 2>/dev/null | wc -c

# Check database slow queries
docker compose exec db mysql -u root -p"..." \
  -e "SHOW PROCESSLIST;"
```

### Disk is full

```bash
# Find large files
du -ah /var/lib/docker/volumes/ | sort -rh | head -20

# Clean old logs
docker system prune

# Clean old backups
find ~/erpnext/backups -mtime +7 -delete
```

### Cannot connect to the database

```bash
# Check if the db container is running
docker compose ps db

# Check the database logs
docker compose logs --tail=50 db

# Common fixes:
docker compose restart db
```

### API errors (500 / 502)

```bash
# Check the backend logs
docker compose logs --tail=100 backend

# Check if the site is configured
docker compose exec backend bench --site erpnext.example.com list-apps

# Restart the backend
docker compose restart backend
```

---

## Emergency procedures

### Server is compromised

1. Immediately stop all containers: `docker compose stop`
2. Change the server root password and all ERPNext passwords
3. Check Docker volumes for unauthorised changes
4. Review the last SSH logins: `last -20`
5. Restore from a clean backup if needed

### Database corrupted

1. Stop all containers: `docker compose stop`
2. Back up the current (corrupted) state: `docker compose volumes backup`
3. Restore from the last known-good backup
4. Run migrate and verify
5. Investigate the root cause before restarting

### Disk full (can't write)

1. Clean Docker logs: `docker system prune -f`
2. Remove old backups: `find ~/erpnext/backups -mtime +7 -delete`
3. Check and remove large files: `du -ah / | sort -rh | head -20`
4. If still full, expand the disk or move backups to remote storage

---

## Key configuration files

| File | Purpose |
|------|---------|
| `.env` | Docker Compose environment variables |
| `config/mariadb-tuning.cnf` | MariaDB performance tuning |
| `config/docker-daemon.json` | Docker log rotation settings |
| `config/logrotate.conf` | System log rotation |
| `config/crontab` | Scheduled tasks (backup, monitoring) |
| `scripts/backup.sh` | Automated backup script |
| `scripts/monitor.sh` | System monitoring script |
