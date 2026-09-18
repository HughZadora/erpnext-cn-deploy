# Deployment guide

> Deploy an ERPNext production environment from scratch. If you are a complete
> beginner, first read the "understand a few concepts" section in the README.

---

## Environment preparation (if this is your first time using Linux)

### Connect to the server

```bash
# Windows users: PuTTY or PowerShell
# Mac/Linux users: terminal
ssh root@your-server-ip
```

### Update the system

```bash
apt update && apt upgrade -y
```

### Install Docker

Docker is the base environment for running ERPNext. One command to install:

```bash
curl -fsSL https://get.docker.com | bash
```

What this command means:
- `curl`: download tool
- `-fsSL`: silent mode, only shows necessary output
- `get.docker.com`: Docker's official installation script
- `| bash`: pipe the downloaded script to bash for execution

After installation:

```bash
# Allow the current user to run docker (avoids typing sudo every time)
sudo usermod -aG docker $USER
```

> ⚠️ **Important**: after running this, exit SSH and log back in, otherwise
> the docker command will report insufficient permissions.

### Verify the installation

```bash
docker --version
# Should show: Docker version 24.x.x or higher

docker compose version
# Should show: Docker Compose version v2.x.x
```

---

## Complete deployment process

### Step 1: Get the project code

```bash
# Create the working directory (all ERPNext-related files go here)
mkdir -p ~/erpnext

# Enter the directory
cd ~/erpnext

# Download frappe_docker from GitHub (the official Docker deployment solution)
git clone https://github.com/frappe/frappe_docker.git

# Enter the frappe_docker directory
cd frappe_docker
```

> 📌 `git clone` downloads the entire project from GitHub to your local machine.

### Step 2: Configure environment variables

```bash
# Copy the official template file
cp example.env .env
```

`example.env` is the official sample file. `cp` makes a copy called `.env`,
which is the configuration file that docker compose actually reads.

Edit `.env`:

```bash
nano .env
```

Find and modify the following (in nano, use arrow keys to move the cursor;
after editing, press Ctrl+X, then Y, then Enter):

```env
# ─── The following two passwords must be changed ───
DB_PASSWORD=your-database-password        # Database root password, use a strong password
ADMIN_PASSWORD=your-admin-password      # ERPNext administrator password

# ─── Modify according to your actual situation ───
SITE_NAME=erpnext.example.com      # Site name; use a domain or any name
FRAPPE_SITE_NAME_HEADER=erpnext.example.com  # Default site
TZ=Asia/Shanghai                   # Time zone; use this for China

# ─── Performance-related; beginners leave these alone ───
GUNICORN_WORKERS=4
GUNICORN_THREADS=4
```

### Step 3: Configure database tuning

MariaDB's default configuration is conservative; we need to optimise it.

```bash
# Go back to the project root
cd ~/erpnext

# Create the MariaDB configuration directory
mkdir -p mariadb-conf
```

Create the configuration file:

```bash
nano mariadb-conf/tuning.cnf
```

Paste the following:

```ini
[mariadb]
innodb_buffer_pool_size=3G
max_connections=200
thread_cache_size=8
slow_query_log=1
long_query_time=1
innodb_log_file_size=512M
innodb_flush_log_at_trx_commit=2
innodb_flush_method=O_DIRECT
tmp_table_size=64M
max_heap_table_size=64M
```

### Step 4: LXC adaptation (determine whether needed)

```bash
cat /sys/kernel/security/apparmor/enabled
```

Three possible outcomes:

| Result | Meaning | Action |
|--------|---------|--------|
| Shows `1` | AppArmor is working normally | Skip this step |
| Shows `0` | AppArmor is disabled | **Adaptation required** |
| Shows `No such file or directory` | No AppArmor | **Adaptation required** |

**If adaptation is needed**, create
`~/erpnext/frappe_docker/overrides/compose.lxc.yaml`:

```bash
nano ~/erpnext/frappe_docker/overrides/compose.lxc.yaml
```

Then open `~/erpnext/frappe_docker/.env` and add at the end:

```env
COMPOSE_FILE=compose.yaml:overrides/compose.mariadb.yaml:overrides/compose.redis.yaml:overrides/compose.noproxy.yaml:overrides/compose.lxc.yaml
```

### Step 5: Start all services

```bash
cd ~/erpnext/frappe_docker

# Start (the -d flag runs in the background)
docker compose up -d
```

The first run downloads images, with output like:

```
[+] Running 11/11
 ✔ frappe_docker Pulled                ← image download complete
 ✔ Container erpnext-db-1 Started      ← database container started
 ✔ Container erpnext-redis-cache-1 Started
 ✔ Container erpnext-redis-queue-1 Started
 ✔ Container erpnext-frontend-1 Started
 ✔ Container erpnext-backend-1 Started
 ...
```

> **If you get an error here** (such as an AppArmor error), check whether the
> LXC adaptation in Step 4 was configured correctly.

After starting, wait 1-2 minutes for the database to finish initialising.

```bash
# Check all container states
docker compose ps
```

All containers should be `Up`. If any show `Exit` or `Restarting`:

```bash
# Check the specific container's logs for troubleshooting
docker compose logs --tail=50 <container-name>
# For example: docker compose logs --tail=50 db
```

### Step 6: Create the site

A "site" is one independent ERP instance.

```bash
cd ~/erpnext/frappe_docker

docker compose exec backend \
  bench new-site erpnext.example.com \
  --admin-password "your-admin-password" \
  --db-root-password "your-db-root-password"
```

Breaking down this command:

| Part | Description |
|------|-------------|
| `docker compose exec backend` | Enter the running backend container |
| `bench new-site` | Use the bench tool to create a new site |
| `erpnext.example.com` | Site name |
| `--admin-password` | Set the Administrator password |
| `--db-root-password` | Connect to the database as root |

If successful, it prints `Site erpnext.example.com has been created`.

**If it fails**, the most common cause is the database not having finished
initialising; wait a minute and try again.

### Step 7: Install the ERPNext application

After creating the site, the site is empty. You need to install the ERPNext
app for functionality.

```bash
# Install ERPNext (the most time-consuming step, 5-10 minutes)
docker compose exec backend bench --site erpnext.example.com install-app erpnext
```

This command will:
1. Download the Python packages that ERPNext depends on
2. Install all modules (finance, sales, inventory, HR...)
3. Create database tables
4. Initialise system data

Do not interrupt during this; you can do other things while waiting.

After installation:

```bash
# Database migration (ensures the table structure is up to date)
docker compose exec backend bench --site erpnext.example.com migrate

# Clear cache
docker compose exec backend bench --site erpnext.example.com clear-cache

# Health check
docker compose exec backend bench --site erpnext.example.com doctor
```

`doctor` checks the system status. If there are no ERROR-level reports, the
installation was successful.

### Step 8: Acceptance

```bash
# Command-line check
curl -s -o /dev/null -w "HTTP status: %{http_code}\n" http://localhost/
# Should normally return 200 or 302
```

Open a browser and visit:

```
http://your-server-ip/login
```

Enter:
- Username: `Administrator`
- Password: the administrator password you set

If everything is normal, you will see the ERPNext main interface. 👏

---

## Follow-up configuration

### Configure backups

```bash
# Copy the backup script from this repository
cp ~/erpnext-deploy/scripts/backup.sh ~/erpnext/backup.sh

# Create the log directory
mkdir -p ~/erpnext/logs

# Test the backup
bash ~/erpnext/backup.sh

# Configure daily automatic backup (2:00 AM)
echo '0 2 * * * root /root/erpnext/backup.sh > /root/erpnext/logs/backup.log 2>&1' \
  | sudo tee /etc/cron.d/erpnext
```

### Configure log rotation

Prevent Docker logs from filling the disk:

```bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

> ⚠️ Restarting Docker restarts all containers; do this in a maintenance window.

### Configure container resource limits

Refer to this repository's `config/compose.lxc.yaml`; adjust CPU and memory
limits according to your server configuration.

---

## Common issues per step

### Step 5 (docker compose up -d) fails

```
Error response from daemon: unable to apply apparmor profile
```
→ Check whether the LXC adaptation (Step 4) was done.

```
port is already allocated
```
→ Port 80 is in use. Set `HTTP_PUBLISH_PORT=8080` in `.env`.

```
Cannot connect to the Docker daemon
```
→ Docker is not running. Run `systemctl start docker`.

### Step 6 (create site) fails

```
Can't connect to MySQL server on 'db'
```
→ The database is not ready yet. Wait 1-2 minutes and try again.

```
socket.gaierror: [Errno -2] Name or service not known
```
→ The backend container cannot resolve the `db` hostname. Restart the
container:
```bash
docker compose restart backend
```

### Step 7 (install-app) fails

```
ModuleNotFoundError: No module named '...'
```
→ A network issue caused dependency download failure; retry.

Installation interrupted by network loss or timeout:
```bash
# Retry the installation
docker compose exec backend bench --site erpnext.example.com install-app erpnext
```

---

## Deployment checklist

- [ ] Docker installed and running normally
- [ ] Passwords in `.env` changed
- [ ] LXC adaptation handled (if needed)
- [ ] MariaDB tuning configuration in place
- [ ] `docker compose up -d` succeeded, all containers `Up`
- [ ] Site created successfully
- [ ] ERPNext installed, `doctor` has no ERROR
- [ ] Website accessible, can log in
- [ ] Backup script executable
- [ ] Scheduled backup configured
- [ ] Log rotation in effect
