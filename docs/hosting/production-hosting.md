# Production Hosting Guide

Step-by-step: going from a fresh VPS to a live Janeway site with SSL.

---

## 1. Pick a Server

Any VPS with at least **1 CPU, 2GB RAM, 40GB disk** works fine to start. More storage if you expect lots of PDF uploads.

Recommended providers:
- **DigitalOcean** — $12/mo (2GB), great docs
- **Hetzner** — $4-8/mo, cheapest
- **Linode / Vultr** — $5-12/mo

OS: **Ubuntu 22.04 LTS** (most guides and packages assume Ubuntu/Debian).

---

## 2. Initial Server Setup

```bash
# SSH in as root
ssh root@your-server-ip

# Create a non-root user
adduser janeway
usermod -aG sudo janeway

# Set up SSH key auth for the new user (from your local machine)
# ssh-copy-id janeway@your-server-ip

# Basic hardening
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable

# Update packages
apt update && apt upgrade -y

# Install essentials
apt install -y git python3 python3-pip python3-venv \
  postgresql postgresql-contrib \
  nginx certbot python3-certbot-nginx \
  build-essential libpq-dev libjpeg-dev zlib1g-dev \
  libxml2-dev libxslt1-dev
```

---

## 3. Set Up PostgreSQL

```bash
sudo -u postgres psql

# Inside psql:
CREATE USER janeway WITH PASSWORD 'use-a-strong-password-here';
CREATE DATABASE janeway OWNER janeway;
GRANT ALL PRIVILEGES ON DATABASE janeway TO janeway;
\q
```

---

## 4. Clone and Configure Janeway

```bash
# Switch to your janeway user
su - janeway

# Clone your fork
git clone https://github.com/YOUR_USERNAME/HeeroPublishing.git
cd HeeroPublishing

# Create a virtualenv
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn psycopg2-binary
```

### Create settings.py

```bash
cp src/core/example_settings.py src/core/settings.py
```

Edit `src/core/settings.py` with these critical changes:

```python
import os

# SECURITY
DEBUG = False
SECRET_KEY = "generate-a-50-char-random-string-here"
# Generate one with: python3 -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"

ALLOWED_HOSTS = ["yourdomain.com", "www.yourdomain.com"]

# DATABASE
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "janeway",
        "USER": "janeway",
        "PASSWORD": "use-a-strong-password-here",
        "HOST": "localhost",
        "PORT": "5432",
    }
}

# URL CONFIG
# "path" = single domain (yourdomain.com/journalcode/)
# "domain" = each journal gets its own domain
URL_CONFIG = "path"

# STATIC/MEDIA FILES
STATIC_ROOT = os.path.join(BASE_DIR, "collected-static")

# EMAIL — pick a provider (Brevo, SendGrid, Mailgun, etc.)
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.your-provider.com"
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = "your-smtp-user"
EMAIL_HOST_PASSWORD = "your-smtp-password"
DEFAULT_FROM_EMAIL = "noreply@yourdomain.com"

# BACKUPS
BACKUP_TYPE = "directory"
BACKUP_DIR = "/backups/janeway/"
BACKUP_EMAIL = True
```

### Initialize the application

```bash
cd /home/janeway/HeeroPublishing

# Set the settings module
export JANEWAY_SETTINGS_MODULE=core.settings

# Run migrations
python src/manage.py migrate

# Collect static files
python src/manage.py collectstatic --noinput

# Create your admin/press (interactive)
python src/manage.py install_janeway

# Test it works (Ctrl+C to stop)
python src/manage.py runserver 0.0.0.0:8000
```

Visit `http://your-server-ip:8000` to verify. Then stop it — we'll use gunicorn instead.

---

## 5. Set Up Gunicorn

Create a systemd service so gunicorn starts automatically and restarts on crash.

```bash
sudo nano /etc/systemd/system/janeway.service
```

```ini
[Unit]
Description=Janeway Gunicorn Daemon
After=network.target postgresql.service

[Service]
User=janeway
Group=janeway
WorkingDirectory=/home/janeway/HeeroPublishing/src
Environment="JANEWAY_SETTINGS_MODULE=core.settings"
Environment="PYTHONPATH=/home/janeway/HeeroPublishing/src"
ExecStart=/home/janeway/HeeroPublishing/venv/bin/gunicorn \
    core.wsgi:application \
    --bind 127.0.0.1:8000 \
    --workers 3 \
    --timeout 120 \
    --access-logfile /var/log/janeway/access.log \
    --error-logfile /var/log/janeway/error.log
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Create log directory
sudo mkdir -p /var/log/janeway
sudo chown janeway:janeway /var/log/janeway

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable janeway
sudo systemctl start janeway

# Check it's running
sudo systemctl status janeway
```

**Workers:** Rule of thumb is `(2 * CPU cores) + 1`. For a 1-core VPS, 3 workers is fine.

---

## 6. Set Up Nginx

```bash
sudo nano /etc/nginx/sites-available/janeway
```

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    client_max_body_size 100M;

    location /static/ {
        alias /home/janeway/HeeroPublishing/src/collected-static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /home/janeway/HeeroPublishing/src/media/;
        expires 7d;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/janeway /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default  # remove default site

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

Visit `http://yourdomain.com` — you should see Janeway.

---

## 7. SSL with Let's Encrypt

Point your domain's DNS A record to your server IP first. Then:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot will automatically:
- Get a free SSL certificate
- Configure nginx to redirect HTTP to HTTPS
- Set up auto-renewal

Verify auto-renewal works:

```bash
sudo certbot renew --dry-run
```

After SSL is working, update `settings.py`:

```python
# Add these for HTTPS
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

Then restart gunicorn:

```bash
sudo systemctl restart janeway
```

---

## 8. Set Up Backups

See [database-backups.md](database-backups.md) for full details. Quick version:

```bash
# Create backup directory
sudo mkdir -p /backups/janeway
sudo chown janeway:janeway /backups/janeway

# Create backup script
sudo nano /opt/janeway-backup.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/backups/janeway"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
JANEWAY_DIR="/home/janeway/HeeroPublishing"
KEEP_DAYS=30

mkdir -p "$BACKUP_DIR"

pg_dump -U janeway -h localhost -d janeway -Fc \
  -f "$BACKUP_DIR/db_$TIMESTAMP.dump"

tar -czf "$BACKUP_DIR/files_$TIMESTAMP.tar.gz" \
  "$JANEWAY_DIR/src/files/" \
  "$JANEWAY_DIR/src/media/"

find "$BACKUP_DIR" -name "*.dump" -mtime +$KEEP_DAYS -delete
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$KEEP_DAYS -delete
```

```bash
sudo chmod +x /opt/janeway-backup.sh

# Add to cron — daily at 2 AM
sudo crontab -u janeway -e
# Add: 0 2 * * * /opt/janeway-backup.sh >> /var/log/janeway-backup.log 2>&1
```

---

## 9. Deploying Updates

When you've pulled upstream changes or made your own:

```bash
# SSH into server
ssh janeway@your-server-ip
cd HeeroPublishing

# Activate virtualenv
source venv/bin/activate
export JANEWAY_SETTINGS_MODULE=core.settings

# Pull changes
git pull origin master

# Install any new dependencies
pip install -r requirements.txt

# Apply database migrations
python src/manage.py migrate

# Collect new static files
python src/manage.py collectstatic --noinput

# Restart gunicorn
sudo systemctl restart janeway
```

---

## 10. Ongoing Maintenance

### Regular tasks

| Task | Frequency | Command |
|------|-----------|---------|
| Check backups are running | Weekly | `ls -la /backups/janeway/` |
| Check disk space | Monthly | `df -h` |
| Update Ubuntu packages | Monthly | `sudo apt update && sudo apt upgrade` |
| Check logs for errors | As needed | `sudo journalctl -u janeway --since "1 hour ago"` |
| Renew SSL cert | Automatic | `sudo certbot renew --dry-run` to verify |
| Sync upstream code | When needed | See setup-guide.md section 3 |

### Useful commands

```bash
# View gunicorn logs
sudo journalctl -u janeway -f              # live tail
sudo journalctl -u janeway --since today    # today's logs

# Restart services
sudo systemctl restart janeway              # gunicorn
sudo systemctl restart nginx                # nginx
sudo systemctl restart postgresql           # database

# Django management
cd /home/janeway/HeeroPublishing
source venv/bin/activate
export JANEWAY_SETTINGS_MODULE=core.settings
python src/manage.py shell                  # Django shell
python src/manage.py createsuperuser        # new admin user
python src/manage.py migrate --list         # see migration status
```

### If something breaks

1. **Site shows 502 Bad Gateway** — gunicorn isn't running. Check `sudo systemctl status janeway` and logs.
2. **Static files missing (unstyled page)** — run `python src/manage.py collectstatic --noinput` and check nginx static config.
3. **Database connection error** — check PostgreSQL is running: `sudo systemctl status postgresql`.
4. **Out of disk space** — check `/backups/`, clear old backups or increase disk size.
5. **After upstream merge, migration errors** — check `python src/manage.py showmigrations` for unapplied migrations.

---

## Checklist: Zero to Live

- [ ] VPS provisioned with Ubuntu 22.04
- [ ] Firewall configured (SSH, 80, 443)
- [ ] PostgreSQL installed and database created
- [ ] Janeway cloned and dependencies installed
- [ ] `settings.py` configured (SECRET_KEY, DB, ALLOWED_HOSTS, EMAIL)
- [ ] `migrate` and `install_janeway` run successfully
- [ ] Gunicorn systemd service running
- [ ] Nginx configured and serving the site
- [ ] DNS pointing to server
- [ ] SSL certificate installed via Let's Encrypt
- [ ] HTTPS settings enabled in `settings.py`
- [ ] Automated daily backups configured
- [ ] Backup tested (restore to a test environment)
- [ ] First journal created via `/manager/`
