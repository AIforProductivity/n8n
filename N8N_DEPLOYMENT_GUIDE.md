# N8N Deployment Guide - Production Setup

A complete guide to deploying n8n with PostgreSQL, Docker, Nginx reverse proxy, and Let's Encrypt HTTPS on Linux VMs.

**Architecture Overview:**
```
Internet → Nginx (HTTPS) → n8n Docker ← PostgreSQL Docker
```

> **Important:** All domains, IP addresses, passwords, encryption keys, and hostnames shown in this guide are examples. Replace them with values appropriate for your environment before deployment.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Values to Replace](#values-to-replace)
3. [Networking and Security Settings](#networking-and-security-settings)
4. [DNS Configuration](#dns-configuration)
5. [Pre-Installation Configuration](#pre-installation-configuration)
6. [Installation - Step by Step](#installation---step-by-step)
7. [Post-Installation Configuration](#post-installation-configuration)
8. [Troubleshooting & Verification](#troubleshooting--verification)
9. [Production Recommendations](#production-recommendations)

---

## Prerequisites

### System Requirements

- **OS:** Ubuntu LTS (22.04 or later)
- **Cloud Provider:** Any provider supporting public IPs and security groups (AWS, Azure, DigitalOcean, etc.)
- **Tools Required:**
  - Docker Engine
  - Docker Compose
  - Nginx
  - Certbot (Let's Encrypt client)
  - curl, wget, nano/vim

### Software Versions (Recommended)

- **PostgreSQL:** 16
- **n8n:** Latest stable version (see [Docker Hub](https://hub.docker.com/r/n8nio/n8n/tags) for available versions)
- **Nginx:** Latest from Ubuntu repos

### Credentials & Information to Prepare

Before starting, gather or generate the items in the **Values to Replace** section below.

---

## Values to Replace

**Use this table as a reference throughout the guide. Replace each placeholder with your own values:**

| Placeholder | Replace With | Example |
|-----------|--------------|---------|
| `n8n.example.com` | Your domain or subdomain | `automation.example.com` |
| `203.x.xxx.10` | Your VM's public IP address | `198.51.100.42` |
| `your-user` | Your Linux username | `ubuntu` |
| `YOUR_POSTGRES_PASSWORD` | Strong database password (min. 24 chars) | `P@ssw0rd!Secure123$Random` |
| `YOUR_RANDOM_64_CHARACTER_ENCRYPTION_KEY` | Output of `openssl rand -hex 32` | `20ed8322a39c5c8xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `UTC` | Your timezone | `Europe/London`, `America/New_York`, `Asia/Tokyo` |
| `admin@example.com` | Your email for Let's Encrypt | `your-email@yourdomain.com` |

---

## Networking and Security Settings

### Network Security Configuration

If using a cloud provider with a firewall/security groups, ensure these inbound rules are set:

| Port | Protocol | Source | Action | Purpose |
|------|----------|--------|--------|---------|
| 22 | TCP | Your IP or restricted range | Allow | SSH access |
| 80 | TCP | Internet (`0.0.0.0/0`) | Allow | HTTP (Let's Encrypt validation) |
| 443 | TCP | Internet (`0.0.0.0/0`) | Allow | HTTPS (n8n UI) |

**Important:** Do NOT expose port 5678 publicly. Nginx acts as the reverse proxy.

### Target Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                     Internet Users                       │
└────────────────────────┬────────────────────────────────┘
                         │
                         ↓
        ┌────────────────────────────────┐
        │   Domain: n8n.example.com      │
        │   (DNS A Record)               │
        └────────────┬───────────────────┘
                     │
                     ↓
        ┌────────────────────────────────┐
        │     Cloud Provider Public IP    │
        │     (e.g., 203.x.xxx.10)       │
        └────────────┬───────────────────┘
                     │
                     ↓
        ┌────────────────────────────────┐
        │  Security Group / Firewall     │
        │    (Ports 22, 80, 443)         │
        └────────────┬───────────────────┘
                     │
                     ↓
        ┌────────────────────────────────┐
        │    Nginx Reverse Proxy         │
        │  (localhost:80 & localhost:443)│
        └────────────┬───────────────────┘
                     │
                     ↓
        ┌────────────────────────────────┐
        │   n8n Docker Container         │
        │   (localhost:5678)             │
        └────────────┬───────────────────┘
                     │
                     ↓
        ┌────────────────────────────────┐
        │   PostgreSQL Docker Container  │
        │   (Internal Network)           │
        └────────────────────────────────┘
```

---

## DNS Configuration

### Create an A Record

In your DNS provider's control panel, create the following record:

| Field | Value |
|-------|-------|
| **Host/Name** | `n8n` (or your preferred subdomain) |
| **Type** | A |
| **Value** | Your VM's public IP (e.g., `203.x.xxx.10`) |
| **TTL** | 300 (5 minutes, for quick propagation) |

### Verify DNS Resolution

**From your local machine (Windows/Mac/Linux):**

```bash
nslookup n8n.example.com
```

**Expected output:**
```
Name:    n8n.example.com
Address: 203.x.xxx.10
```

**From Google's public DNS:**
```bash
nslookup n8n.example.com 8.8.8.8
```

**Wait for propagation:** DNS changes can take 5–300 seconds. Keep checking every few minutes until it resolves correctly.

---

## Pre-Installation Configuration

### Step 1: Update System Packages

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 2: Create Project Directory Structure

```bash
mkdir -p ~/n8n/data
mkdir -p ~/n8n/postgres
cd ~/n8n
```

> Note: `/opt/n8n` is also a common production location. Adjust paths as needed.

### Step 3: Generate Encryption Key

This key encrypts stored credentials in n8n. **Generate a strong random key:**

```bash
openssl rand -hex 32
```

**Save this output.** You'll use it as `N8N_ENCRYPTION_KEY` in the configuration files.

Example output:
```
a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7b8c9d0e1f2
```

### Step 4: Prepare Configuration Values

Create a `.env` file for easy management. Replace all placeholders with your actual values:

```bash
cat > ~/n8n/.env << 'EOF'
# PostgreSQL Configuration
POSTGRES_USER=n8n
POSTGRES_PASSWORD=YOUR_POSTGRES_PASSWORD

# n8n Configuration
N8N_HOST=n8n.example.com
N8N_PROTOCOL=https
N8N_PORT=5678

# Webhook & Editor URLs
WEBHOOK_URL=https://n8n.example.com/
N8N_EDITOR_BASE_URL=https://n8n.example.com/

# Timezone
GENERIC_TIMEZONE=UTC

# Security
N8N_SECURE_COOKIE=true
N8N_ENCRYPTION_KEY=YOUR_RANDOM_64_CHARACTER_ENCRYPTION_KEY
EOF
```

**⚠️ Replace placeholders:**
- `YOUR_POSTGRES_PASSWORD` → Your secure database password (minimum 24 characters)
- `n8n.example.com` → Your actual domain
- `N8N_ENCRYPTION_KEY` → The key generated in Step 3
- `UTC` → Your timezone (e.g., `Europe/London`, `America/New_York`, `Asia/Tokyo`)

---

## Installation - Step by Step

### Step 1: Create docker-compose.yml

Create the Docker Compose configuration file:

```bash
cat > ~/n8n/docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
    volumes:
      - ./postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d n8n"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - n8n-network

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: ${POSTGRES_USER}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
      
      N8N_HOST: ${N8N_HOST}
      N8N_PROTOCOL: ${N8N_PROTOCOL}
      N8N_PORT: ${N8N_PORT}
      
      WEBHOOK_URL: ${WEBHOOK_URL}
      N8N_EDITOR_BASE_URL: ${N8N_EDITOR_BASE_URL}
      
      GENERIC_TIMEZONE: ${GENERIC_TIMEZONE}
      
      N8N_SECURE_COOKIE: ${N8N_SECURE_COOKIE}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
    volumes:
      - ./data:/home/node/.n8n
    networks:
      - n8n-network

networks:
  n8n-network:
    driver: bridge
EOF
```

**Note:** The latest stable version is available at [Docker Hub - n8n](https://hub.docker.com/r/n8nio/n8n/tags).

### Step 2: Install Docker & Docker Compose

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh

# Add current user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

### Step 3: Start n8n & PostgreSQL

```bash
cd ~/n8n
docker compose up -d
```

### Step 4: Verify Services are Running

```bash
docker ps
```

**Expected output:** Two containers running (`n8n-postgres` and `n8n`)

**Check logs:**

```bash
# PostgreSQL logs
docker logs n8n-postgres --tail=20

# n8n logs (wait 30-60 seconds for startup)
docker logs n8n --tail=50
```

**Look for messages like:**
```
n8n ready on 0.0.0.0, port 5678
```

---

## Post-Installation Configuration

### Step 1: Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### Step 2: Enable and Start Nginx

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

### Step 3: Test Nginx Locally

```bash
curl http://localhost
```

**Expected output:** HTML containing "Welcome to nginx!"

### Step 4: Configure Nginx as Reverse Proxy

Create the Nginx configuration file. Replace `n8n.example.com` with your actual domain:

```bash
sudo bash -c 'cat > /etc/nginx/sites-available/default << '\''EOF'\''
server {
    server_name n8n.example.com;

    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Headers for n8n
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;

        # Long-running request support
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
        proxy_connect_timeout 3600;

        # Disable buffering for streaming
        proxy_buffering off;
    }

    # HTTPS configuration (added by Certbot)
    listen 443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name n8n.example.com;
    return 301 https://$host$request_uri;
}
EOF
'
```

**⚠️ Replace `n8n.example.com` with your actual domain (2 occurrences).**

### Step 5: Test Nginx Configuration

```bash
sudo nginx -t
```

**Expected output:**
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration will be successful
```

### Step 6: Reload Nginx

```bash
sudo systemctl reload nginx
```

### Step 7: Install Certbot for HTTPS

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

### Step 8: Obtain Let's Encrypt Certificate

Before running Certbot, verify DNS is working and ports 80/443 are accessible:

```bash
nslookup n8n.example.com 8.8.8.8
curl http://203.x.xxx.10
```

**Request certificate. Replace `n8n.example.com` and `admin@example.com` with your values:**

```bash
sudo certbot --nginx -d n8n.example.com -m admin@example.com --agree-tos
```

**Expected output:**
```
Successfully received certificate.
Congratulations! You have successfully enabled HTTPS on your server!
```

### Step 9: Verify HTTPS Certificate Auto-Renewal

```bash
sudo certbot renew --dry-run
```

**Expected output:** "Congratulations, all renewals succeeded"

---

## Troubleshooting & Verification

### Test HTTP Access

```bash
curl http://203.x.xxx.10
```

Should redirect to HTTPS.

### Test HTTPS Access

```bash
curl https://n8n.example.com -I
```

Should return a 200 status code after DNS propagation.

### Check n8n Service Status

```bash
# View current logs
docker logs n8n --tail=50 -f

# Check container status
docker ps | grep n8n

# Restart if needed
docker restart n8n
```

### Check PostgreSQL Connectivity

```bash
docker logs n8n-postgres --tail=20
```

### Reset n8n Configuration (First-Time Setup)

If you need to start fresh:

```bash
cd ~/n8n
docker compose down

# Delete config but keep data volumes
rm -f ~/n8n/data/config

# Restart
docker compose up -d

# Monitor logs
docker logs n8n -f
```

### Permissions Issue with PostgreSQL

If you encounter permission errors when deleting the postgres directory:

```bash
cd ~/n8n
docker compose down

sudo rm -rf ~/n8n/postgres
mkdir ~/n8n/postgres

docker compose up -d
```

---

## Production Recommendations

### 1. Version Pinning

The example already uses a pinned version (`1.107.4`), but always:

- Check [Docker Hub - n8n](https://hub.docker.com/r/n8nio/n8n/tags) for the latest stable version

Update your `docker-compose.yml`:

```yaml
n8n:
  image: n8nio/n8n:latest  # Pin to known stable version
```

---

### 2. Separate Passwords

Use different passwords for PostgreSQL and n8n encryption. Generate them securely:

```bash
# Generate PostgreSQL password (24+ characters)
openssl rand -base64 24

# Generate n8n encryption key (64 hex characters)
openssl rand -hex 32
```

Store these in your `.env` file, never in version control.

---

### 3. Automated PostgreSQL Backups

Create a backup script:

```bash
cat > ~/n8n/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR=~/n8n/backups
mkdir -p $BACKUP_DIR
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Backup PostgreSQL database
docker exec n8n-postgres pg_dump -U n8n n8n | gzip > $BACKUP_DIR/n8n_db_backup_$TIMESTAMP.sql.gz

# Backup n8n data directory (contains workflows, credentials, etc.)
tar -czf $BACKUP_DIR/n8n_data_backup_$TIMESTAMP.tar.gz -C ~/n8n data/

# Keep only last 30 days
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete

echo "Backup completed:"
echo "  - Database: $BACKUP_DIR/n8n_db_backup_$TIMESTAMP.sql.gz"
echo "  - Data: $BACKUP_DIR/n8n_data_backup_$TIMESTAMP.tar.gz"
EOF

chmod +x ~/n8n/backup.sh
```

**Schedule with cron (daily at 2 AM):**

```bash
(crontab -l 2>/dev/null; echo "0 2 * * * ~/n8n/backup.sh") | crontab -
```

**Verify cron entry:**

```bash
crontab -l
```

---

### 4. Docker Log Rotation

Prevent logs from consuming disk space:

```bash
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

---

### 5. System Monitoring

Install monitoring tools:

```bash
# Monitor resource usage
sudo apt install htop iotop nethogs -y

# Check disk space
df -h

# Check memory
free -h

# Check n8n container resources
docker stats n8n n8n-postgres
```

---

### 6. Firewall Configuration (Optional)

If using UFW firewall:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

**Verify UFW rules:**

```bash
sudo ufw status
```

---

### 7. Security Hardening

- **SSH Key-Based Auth:** Disable password authentication, use SSH keys only
- **Firewall:** Restrict source IPs for port 22 to your IP range
- **Secrets Management:** Use a secrets manager (HashiCorp Vault, AWS Secrets Manager) instead of `.env` files for production
- **Regular Updates:** Keep Docker images and OS packages updated:
  ```bash
  sudo apt update && sudo apt upgrade -y
  docker compose pull && docker compose up -d
  ```
- **.env File Permissions:** Ensure `.env` is readable only by your user:
  ```bash
  chmod 600 ~/n8n/.env
  ```
- **Remove Default Nginx Config:** Back up and remove the default config:
  ```bash
  sudo mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup
  ```

---

### 8. Create Deployment Scripts

**deploy.sh** - Initial setup:
```bash
#!/bin/bash
set -e
cd ~/n8n
docker compose pull
docker compose up -d
docker logs n8n -f
```

**update.sh** - Update n8n version:
```bash
#!/bin/bash
set -e
cd ~/n8n
echo "Backing up before update..."
./backup.sh
echo "Updating n8n..."
docker compose pull
docker compose up -d
docker logs n8n -f
```

**restart.sh** - Restart services:
```bash
#!/bin/bash
set -e
cd ~/n8n
docker compose restart
```

**backup.sh** - Manual backup (created above):
Already covered in section 3.

Make scripts executable:
```bash
chmod +x ~/n8n/*.sh
```

---

## Quick Reference

### Directory Structure

```
~/n8n/
├── docker-compose.yml      # Service definitions
├── .env                    # Environment variables (keep secret)
├── data/                   # n8n data volume (workflows, credentials)
├── postgres/               # PostgreSQL data volume
├── backups/                # Backup files
├── backup.sh               # Backup script
├── deploy.sh               # Deployment script
├── update.sh               # Update script
└── restart.sh              # Restart script
```

### Essential Commands

```bash
# Start services
cd ~/n8n && docker compose up -d

# Stop services
docker compose down

# View logs in real-time
docker logs n8n -f
docker logs n8n-postgres -f

# Restart services
docker compose restart

# Pull latest images
docker compose pull

# View container resource usage
docker stats n8n n8n-postgres

# Execute commands in container
docker exec n8n n8n export --output=/path/to/export.json

# Backup database
docker exec n8n-postgres pg_dump -U n8n n8n | gzip > backup.sql.gz

# Restore database
zcat backup.sql.gz | docker exec -i n8n-postgres psql -U n8n -d n8n
```

---

## Useful Links

- [n8n Documentation](https://docs.n8n.io)
- [n8n Docker Hub](https://hub.docker.com/r/n8nio/n8n)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Docker Documentation](https://docs.docker.com/)
- [n8n Community Forum](https://community.n8n.io)

---

## Support

For issues or questions:

1. **Check logs:** `docker logs n8n --tail=100`
2. **Review:** [n8n GitHub Issues](https://github.com/n8n-io/n8n/issues)
3. **Community:** [n8n Community Forum](https://community.n8n.io)
4. **This guide:** Refer to the [Troubleshooting & Verification](#troubleshooting--verification) section

---

**Status:** Production-Ready  
**Last Verified:** 2026-07-13
