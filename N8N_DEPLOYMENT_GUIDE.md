# N8N Deployment Guide - Production Setup

A comprehensive guide to deploying n8n with PostgreSQL, Docker, Nginx reverse proxy, and Let's Encrypt HTTPS on Linux VMs.

## Architecture Overview

```
Internet → Nginx (HTTPS) → n8n Docker Container ← PostgreSQL Docker Container
```

> **Important:** This guide contains placeholder values. Before deploying, replace all placeholders with actual values specific to your environment. Never use example values in production.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Placeholder Values Reference](#placeholder-values-reference)
3. [Network & Firewall Configuration](#network--firewall-configuration)
4. [DNS Configuration](#dns-configuration)
5. [Pre-Installation Setup](#pre-installation-setup)
6. [Installation - Step by Step](#installation---step-by-step)
7. [Owner Account Setup](#owner-account-setup)
8. [HTTPS Configuration](#https-configuration)
9. [Post-Installation Verification](#post-installation-verification)
10. [Backup & Restore](#backup--restore)
11. [Updates & Upgrades](#updates--upgrades)
12. [Migration to Another VM](#migration-to-another-vm)
13. [Troubleshooting](#troubleshooting)
14. [Security Best Practices](#security-best-practices)
15. [Quick Reference](#quick-reference)

---

## Prerequisites

### System Requirements

- **OS:** Ubuntu LTS (22.04 or later)
- **Cloud Provider:** Any provider supporting public IPs and security groups (AWS, Azure, DigitalOcean, etc.)
- **Tools Required:** curl, wget, nano/vim
- **Permissions:** Sudo access on the VM
- **Network:** Public IP address and domain name

### Software Versions (Recommended)

- **PostgreSQL:** 16
- **n8n:** Current stable version (see [Docker Hub - n8n Tags](https://hub.docker.com/r/n8nio/n8n/tags))
- **Nginx:** Latest from Ubuntu repositories
- **Docker Engine:** Latest from official Docker repository
- **Docker Compose:** v2.x or later

### Information to Prepare

Before starting deployment, gather or generate the following:

1. Your public IP address (from cloud provider console)
2. Your domain name (or subdomain for n8n)
3. Your email address (for Let's Encrypt certificate)
4. Your Linux username
5. Strong passwords for PostgreSQL and n8n encryption key

---

## Placeholder Values Reference

This table documents all placeholder values used throughout this guide. Replace each placeholder with your actual values before executing any commands.

| Placeholder | Purpose | Example | Notes |
|-----------|---------|---------|-------|
| `YOUR_LINUX_USER` | Linux account name | `ubuntu` | User running Docker commands |
| `YOUR_N8N_DOMAIN` | Domain for n8n | `n8n.example.com` | Must resolve to `YOUR_PUBLIC_IP` |
| `YOUR_PUBLIC_IP` | VM's public IP address | `203.0.113.42` | From cloud provider console |
| `YOUR_POSTGRES_PASSWORD` | PostgreSQL database password | (generated) | Minimum 24 characters, use `openssl rand -base64 24` |
| `YOUR_N8N_ENCRYPTION_KEY` | n8n credential encryption key | (generated) | Use `openssl rand -hex 32` (64 hex characters) |
| `YOUR_TIMEZONE` | Server timezone | `UTC`, `America/New_York`, `Europe/London` | See [IANA timezone list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) |
| `YOUR_EMAIL` | Email for Let's Encrypt | `admin@example.com` | Used for certificate renewal notifications |
| `YOUR_N8N_VERSION` | n8n Docker image version | `1.107.4` | Use specific stable version, never `latest` |

**Generate secure passwords:**

```bash
# Generate PostgreSQL password (24+ characters)
openssl rand -base64 24

# Generate n8n encryption key (64 hex characters)
openssl rand -hex 32
```

---

## Network & Firewall Configuration

### Cloud Provider Security Groups

If using AWS, Azure, DigitalOcean, or similar, configure inbound rules:

| Port | Protocol | Source | Action | Purpose |
|------|----------|--------|--------|---------|
| 22 | TCP | Your IP or restricted range | Allow | SSH access |
| 80 | TCP | 0.0.0.0/0 | Allow | HTTP (Let's Encrypt validation) |
| 443 | TCP | 0.0.0.0/0 | Allow | HTTPS (n8n UI) |

**Critical:** Do NOT expose port 5678 publicly. Nginx acts as the reverse proxy and Nginx only listens on localhost:80 and localhost:443.

### Ubuntu UFW Firewall (Optional)

If UFW is enabled on the VM:

```bash
# Enable UFW if not already active
sudo ufw enable

# Allow required ports
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Verify rules
sudo ufw status
```

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                 Internet Users                          │
│             (accessing n8n workflows)                   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ↓
        ┌──────────────────────────────────┐
        │  Domain: YOUR_N8N_DOMAIN         │
        │  (DNS A Record → YOUR_PUBLIC_IP) │
        └──────────────┬───────────────────┘
                       │
                       ↓
        ┌──────────────────────────────────┐
        │  Cloud Provider Public IP        │
        │  (YOUR_PUBLIC_IP)                │
        └──────────────┬───────────────────┘
                       │
                       ↓
        ┌──────────────────────────────────┐
        │  Security Group / UFW Firewall   │
        │  (Ports: 22, 80, 443)            │
        └──────────────┬───────────────────┘
                       │
                       ↓
        ┌──────────────────────────────────┐
        │   Nginx Reverse Proxy            │
        │  (localhost:80 & localhost:443)  │
        └──────────────┬───────────────────┘
                       │
                       ↓
        ┌──────────────────────────────────┐
        │  n8n Docker Container            │
        │  (localhost:5678, not exposed)   │
        └──────────────┬───────────────────┘
                       │
                       ↓
        ┌──────────────────────────────────┐
        │  PostgreSQL Docker Container     │
        │  (Internal Docker network)       │
        └──────────────────────────────────┘
```

---

## DNS Configuration

### Create DNS A Record

In your DNS provider's control panel (Route53, Cloudflare, Namecheap, etc.):

| Field | Value |
|-------|-------|
| **Hostname/Name** | `n8n` (or your preferred subdomain prefix) |
| **Type** | A |
| **Value** | `YOUR_PUBLIC_IP` |
| **TTL** | 300 (5 minutes for quick propagation) |

### Verify DNS Resolution

Test DNS propagation from your local machine:

```bash
# Query your default resolver
nslookup YOUR_N8N_DOMAIN

# Query Google's public DNS (8.8.8.8)
nslookup YOUR_N8N_DOMAIN 8.8.8.8

# Alternative: use dig
dig YOUR_N8N_DOMAIN

# Alternative: use host command
host YOUR_N8N_DOMAIN
```

Expected output:
```
YOUR_N8N_DOMAIN has address YOUR_PUBLIC_IP
```

**DNS Propagation:** Changes can take 5 seconds to 300+ seconds depending on TTL. Repeatedly check until the record resolves correctly before proceeding to HTTPS setup.

---

## Pre-Installation Setup

### Step 1: Update System Packages

Connect to your VM and update all packages:

```bash
ssh YOUR_LINUX_USER@YOUR_PUBLIC_IP

# Update package manager
sudo apt update
sudo apt upgrade -y

# Verify system is ready
uname -a
```

**Expected output:** Ubuntu version information confirming OS is up to date.

### Step 2: Create Application Directory

Create the standardized application directory structure:

```bash
# Create main application directory
mkdir -p ~/Apps/n8n/data
mkdir -p ~/Apps/n8n/postgres
mkdir -p ~/Apps/n8n/backups

# Navigate to application directory
cd ~/Apps/n8n

# Verify directory structure
ls -la ~/Apps/n8n
```

**Expected output:**
```
drwxr-xr-x  data
drwxr-xr-x  postgres
drwxr-xr-x  backups
```

### Step 3: Generate Secure Passwords

Generate PostgreSQL and n8n encryption keys:

```bash
# Generate PostgreSQL password (24+ characters, base64 encoded)
echo "PostgreSQL Password:"
openssl rand -base64 24

# Generate n8n encryption key (64 hex characters)
echo "n8n Encryption Key:"
openssl rand -hex 32
```

**Save these values securely.** You will need them in the next step. Store them in a password manager or secure note.

### Step 4: Create Environment Configuration File

Create the `.env` file with all configuration values:

```bash
cat > ~/Apps/n8n/.env << 'EOF'
# PostgreSQL Configuration
# Keep these values identical when migrating to another VM
POSTGRES_USER=n8n
POSTGRES_PASSWORD=YOUR_POSTGRES_PASSWORD

# n8n Configuration
# Change these values when moving to a different domain or VM
N8N_HOST=YOUR_N8N_DOMAIN
N8N_PROTOCOL=https
N8N_PORT=5678

# Webhook URLs (must match your domain)
WEBHOOK_URL=https://YOUR_N8N_DOMAIN/
N8N_EDITOR_BASE_URL=https://YOUR_N8N_DOMAIN/

# Timezone
# Set to your server's timezone. See https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
GENERIC_TIMEZONE=YOUR_TIMEZONE

# Security
N8N_SECURE_COOKIE=true
N8N_ENCRYPTION_KEY=YOUR_N8N_ENCRYPTION_KEY

# System limits
N8N_LOG_LEVEL=info
EOF
```

**Secure the .env file:**

```bash
# Make .env readable only by the current user
chmod 600 ~/Apps/n8n/.env

# Verify permissions (should show rw-------)
ls -la ~/Apps/n8n/.env
```

**Important variables:**

- `POSTGRES_PASSWORD` & `N8N_ENCRYPTION_KEY`: Keep identical when migrating. These encrypt your data.
- `N8N_HOST`, `WEBHOOK_URL`, `N8N_EDITOR_BASE_URL`: Change these when deploying to a different domain.
- `GENERIC_TIMEZONE`: Set to your server's timezone.
- `.env` file must never be committed to version control or shared publicly.

---

## Installation - Step by Step

### Step 1: Install Docker Engine

Install Docker from the official repository:

```bash
# Download official Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh

# Execute installation script
sudo sh get-docker.sh

# Remove installation script
rm get-docker.sh

# Add current user to docker group (allows running docker without sudo)
sudo usermod -aG docker $USER

# Apply new group membership (required for immediate effect)
newgrp docker

# Verify Docker installation
docker --version
docker run hello-world
```

**Expected output:**
```
Docker version XX.XX.XX
Hello from Docker!
```

### Step 2: Install Docker Compose

Docker Compose v2 is typically installed with Docker. Verify installation:

```bash
# Check Docker Compose version
docker compose version

# Expected output: Docker Compose version v2.x.x
```

If Docker Compose is not installed, install it manually:

```bash
# Install Docker Compose v2
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Make executable
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker compose version
```

### Step 3: Verify Docker Installation

Before proceeding, verify Docker is properly installed:

```bash
# Check Docker daemon status
sudo systemctl status docker

# List running containers (should be empty initially)
docker ps

# Test Docker functionality
docker run --rm alpine echo "Docker is working!"
```

**Expected output:**
```
Docker daemon active ✓
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS
(no containers yet)
Docker is working!
```

### Step 4: Create Docker Compose Configuration

Create the `docker-compose.yml` file in `~/Apps/n8n/`:

```bash
cat > ~/Apps/n8n/docker-compose.yml << 'EOF'
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
      start_period: 20s
    
    networks:
      - n8n-network

  n8n:
    image: n8nio/n8n:YOUR_N8N_VERSION
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

**Important:** Replace `YOUR_N8N_VERSION` with a specific stable version. Find available versions at [Docker Hub - n8n Tags](https://hub.docker.com/r/n8nio/n8n/tags). Do NOT use `latest` in production.

Modern versions of Docker Compose (Compose V2) no longer require the version: field. If present, Docker displays a warning similar to:

the attribute `version` is obsolete, it will be ignored

This warning does not prevent deployment, but the recommended practice is to omit the version: field and begin the file directly with:

services:

This follows the current Docker Compose specification.


### Step 5: Deploy Containers

Start the n8n and PostgreSQL containers:

```bash
# Navigate to application directory
cd ~/Apps/n8n

# Start containers in background
docker compose up -d

# Wait 30-60 seconds for startup
sleep 45

# Display logs
docker logs n8n --tail=50
```

**Expected output:**
```
Creating n8n-postgres ... done
Creating n8n ... done
```

### Step 6: Verify Container Health

Check that both containers are running and healthy:

```bash
# List running containers
docker ps

# Check container details
docker compose ps
```

**Expected output:**
```
CONTAINER ID  IMAGE               STATUS              NAMES
xxxxxxxx      postgres:16         Up XX seconds       n8n-postgres (healthy)
xxxxxxxx      n8nio/n8n:X.X.X     Up XX seconds       n8n
```

### Step 7: Verify n8n Startup

Check n8n container logs for successful startup:

```bash
# View full startup logs
docker logs n8n --tail=100

# Monitor logs in real-time (press Ctrl+C to exit)
docker logs n8n -f
```

**Look for messages like:**
```
Editor is available at: http://127.0.0.1:5678
```

**If startup is slow or fails:**

```bash
# Check PostgreSQL health
docker logs n8n-postgres --tail=20

# Check for specific errors
docker logs n8n | grep -i "error\|failed"
```

---

## Owner Account Setup

After n8n starts successfully, create the owner account through the web interface.

### Step 1: Verify n8n Is Accessible Locally

Test n8n accessibility on the VM:

```bash
# Test HTTP connectivity
curl -I http://127.0.0.1:5678

# Expected: HTTP/1.1 200 OK
```

### Step 2: Access n8n Web Interface

You have two options:

**Option A: Direct Access (after HTTPS is configured)**

After completing the HTTPS setup in the next section, open your browser and navigate to:

```
https://YOUR_N8N_DOMAIN
```

**Option B: SSH Tunneling (immediate access)**

If HTTPS is not yet configured, use SSH port forwarding:

```bash
# From your local machine
ssh -L 5678:127.0.0.1:5678 YOUR_LINUX_USER@YOUR_PUBLIC_IP

# Then open browser to: http://localhost:5678
```

### Step 3: Create Owner Account

On first access, n8n presents an owner account creation page. Complete the following:

**Required Information:**

- **Email:** Your email address for owner account
- **Password:** Strong password (minimum 8 characters)
- **First Name:** Your first name
- **Last Name:** Your last name

After providing this information, n8n creates the owner account. This account exists only once per deployment and cannot be recreated without resetting the data folder.

### Step 4: Login and Verify

After creating the owner account, log in with the credentials you provided. You should see the n8n editor interface.

**Verification steps:**

```bash
# Check credentials storage
docker exec n8n-postgres psql -U n8n -d n8n -c "SELECT COUNT(*) FROM user;"
```

**Important:** The owner account is tied to the data folder. Deleting `~/Apps/n8n/data` after account creation will force re-initialization and require re-creating the owner account.

---

## HTTPS Configuration

### Step 1: Install Nginx

Install Nginx web server:

```bash
# Update package manager
sudo apt update

# Install Nginx
sudo apt install nginx -y

# Verify installation
nginx -v

# Expected output: nginx version X.XX.X
```

### Step 2: Enable and Start Nginx

Enable Nginx to start on boot and start the service:

```bash
# Enable Nginx to start on system boot
sudo systemctl enable nginx

# Start Nginx service
sudo systemctl start nginx

# Check service status
sudo systemctl status nginx

# Expected output: active (running)
```

### Step 3: Verify Nginx Is Running

Test Nginx locally on the VM:

```bash
# Test HTTP connectivity
curl -I http://127.0.0.1

# Expected: HTTP/1.1 200 OK
```

### Step 4: Configure Nginx as Reverse Proxy

Create Nginx configuration for reverse proxy to n8n:

```bash
# Backup default Nginx config
sudo mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup

# Create new Nginx config
sudo bash -c 'cat > /etc/nginx/sites-available/default << '\''EOF'\''
server {
    server_name YOUR_N8N_DOMAIN;
    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;

        # WebSocket support for real-time updates
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Headers for n8n
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;

        # Support for long-running workflows
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
        proxy_connect_timeout 3600;

        # Disable buffering for streaming responses
        proxy_buffering off;
    }

    # HTTPS configuration (added by Certbot)
    listen 443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/YOUR_N8N_DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/YOUR_N8N_DOMAIN/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name YOUR_N8N_DOMAIN;
    return 301 https://$host$request_uri;
}
EOF
'
```

**Critical:** Replace `YOUR_N8N_DOMAIN` with your actual domain (appears 2 times in the config).

### Step 5: Test Nginx Configuration

Verify the Nginx configuration syntax is correct:

```bash
# Test configuration
sudo nginx -t

# Expected output:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration will be successful
```

### Step 6: Reload Nginx

Load the new configuration:

```bash
# Reload Nginx
sudo systemctl reload nginx

# Verify HTTP redirects to HTTPS (expected: 301)
curl -I http://YOUR_PUBLIC_IP
```

### Step 7: Install Certbot for Let's Encrypt

Install Certbot to manage SSL certificates:

```bash
# Update package manager
sudo apt update

# Install Certbot and Nginx plugin
sudo apt install certbot python3-certbot-nginx -y

# Verify installation
certbot --version

# Expected output: certbot X.XX.X
```

### Step 8: Request Let's Encrypt Certificate

Before requesting a certificate, verify DNS resolution and network connectivity:

```bash
# Verify DNS resolution
nslookup YOUR_N8N_DOMAIN 8.8.8.8

# Expected: YOUR_N8N_DOMAIN has address YOUR_PUBLIC_IP

# Test HTTP access from the internet
# (from your local machine, not the VM)
curl -I http://YOUR_N8N_DOMAIN
```

Request the SSL certificate:

```bash
# Request certificate for your domain
sudo certbot --nginx \
  -d YOUR_N8N_DOMAIN \
  -m YOUR_EMAIL \
  --agree-tos \
  --non-interactive

# Expected output:
# Successfully received certificate.
# Congratulations! You have successfully enabled HTTPS on your server!
```

**If certificate issuance fails:**

```bash
# Check Certbot logs
sudo certbot renew --dry-run --verbose

# Check Nginx error logs
sudo tail -50 /var/log/nginx/error.log

# Verify firewall allows port 80 (required for Let's Encrypt validation)
sudo ufw status numbered | grep 80
```

### Step 9: Verify Certificate Installation

Check that Nginx is now using HTTPS:

```bash
# Test HTTPS certificate
curl -I https://YOUR_N8N_DOMAIN

# Expected: HTTP/1.1 200 OK
# SSL certificate: issued by Let's Encrypt

# Check certificate details
echo | openssl s_client -connect YOUR_N8N_DOMAIN:443 -servername YOUR_N8N_DOMAIN

# Expected: Verify return code: 0 (ok)
```

### Step 10: Enable Certificate Auto-Renewal

Let's Encrypt certificates expire after 90 days. Set up automatic renewal:

```bash
# Test renewal process (dry run)
sudo certbot renew --dry-run --verbose

# Expected output:
# Congratulations, all renewals succeeded.

# Check Certbot renewal timer
sudo systemctl status certbot.timer

# Expected output: active (running)

# View renewal schedule
sudo systemctl list-timers certbot.timer
```

---

## Post-Installation Verification

### Step 1: Verify HTTPS Access

Test HTTPS connectivity from your local machine:

```bash
# From your local machine (not the VM)
curl -I https://YOUR_N8N_DOMAIN

# Expected: HTTP/1.1 200 OK (with SSL certificate info)
```

### Step 2: Access n8n Editor

Open your browser and navigate to:

```
https://YOUR_N8N_DOMAIN
```

You should see the n8n login screen. Log in with the owner account credentials created earlier.

### Step 3: Verify Container Health

Check all containers are healthy:

```bash
# Check container status
docker compose -f ~/Apps/n8n/docker-compose.yml ps

# Expected: All containers showing "Up" status

# Check PostgreSQL health
docker exec n8n-postgres pg_isready -U n8n -d n8n

# Expected: accepting connections
```

### Step 4: Test n8n Functionality

Perform a basic test of n8n:

1. Log in to the n8n editor
2. Create a simple test workflow (e.g., trigger + HTTP request)
3. Execute the workflow to verify database connectivity
4. Save the workflow

### Step 5: Check Logs for Errors

Monitor logs for any issues:

```bash
# Check n8n logs
docker logs n8n --tail=50 | grep -i "error\|warn"

# Check PostgreSQL logs
docker logs n8n-postgres --tail=50 | grep -i "error\|warn"

# Check Nginx logs
sudo tail -50 /var/log/nginx/error.log
```

### Step 6: Document Installation Date

Record the deployment details:

```bash
# Create deployment record
cat > ~/Apps/n8n/DEPLOYMENT_INFO.txt << 'EOF'
N8N Deployment Information
===========================

Deployment Date: $(date)
n8n Version: YOUR_N8N_VERSION
PostgreSQL Version: 16
Domain: YOUR_N8N_DOMAIN
IP Address: YOUR_PUBLIC_IP
Timezone: YOUR_TIMEZONE
Nginx: Running
Let's Encrypt: Active
PostgreSQL Status: Running

Owner Account Email: (as configured during setup)

Last Updated: $(date)
EOF

cat ~/Apps/n8n/DEPLOYMENT_INFO.txt
```

---

## Backup & Restore

### Database Backup

#### Manual Database Backup

Create a backup of the PostgreSQL database:

```bash
# Backup PostgreSQL database (SQL format)
docker exec n8n-postgres pg_dump -U n8n -d n8n > ~/Apps/n8n/backups/n8n_db_backup_$(date +%Y%m%d_%H%M%S).sql

# Backup PostgreSQL database (compressed format, recommended)
docker exec n8n-postgres pg_dump -U n8n -d n8n | gzip > ~/Apps/n8n/backups/n8n_db_backup_$(date +%Y%m%d_%H%M%S).sql.gz

# Verify backup file
ls -lh ~/Apps/n8n/backups/
```

#### Automated Database Backup Script

Create a script for automated backups:

```bash
cat > ~/Apps/n8n/backup_database.sh << 'EOF'
#!/bin/bash

set -e

BACKUP_DIR=~/Apps/n8n/backups
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Backup PostgreSQL database
echo "Backing up PostgreSQL database..."
docker exec n8n-postgres pg_dump -U n8n -d n8n | gzip > "$BACKUP_DIR/n8n_db_backup_$TIMESTAMP.sql.gz"

echo "Database backup completed: $BACKUP_DIR/n8n_db_backup_$TIMESTAMP.sql.gz"

# Remove backups older than RETENTION_DAYS
echo "Cleaning up old backups (older than $RETENTION_DAYS days)..."
find "$BACKUP_DIR" -name "n8n_db_backup_*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup retention policy applied."
EOF

# Make script executable
chmod +x ~/Apps/n8n/backup_database.sh

# Test the script
~/Apps/n8n/backup_database.sh
```

Schedule automated backups with cron (daily at 2:00 AM):

```bash
# Edit crontab
crontab -e

# Add the following line:
# 0 2 * * * ~/Apps/n8n/backup_database.sh

# Verify cron entry
crontab -l
```

### Data Backup (Workflows & Credentials)

#### Manual Data Backup

Create a backup of n8n data directory:

```bash
# Backup n8n data directory (contains workflows, credentials, settings)
tar -czf ~/Apps/n8n/backups/n8n_data_backup_$(date +%Y%m%d_%H%M%S).tar.gz \
  -C ~/Apps/n8n data/

# Verify backup
ls -lh ~/Apps/n8n/backups/n8n_data_backup_*.tar.gz
```

#### Automated Data Backup Script

Create a script for automated data backups:

```bash
cat > ~/Apps/n8n/backup_data.sh << 'EOF'
#!/bin/bash

set -e

BACKUP_DIR=~/Apps/n8n/backups
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Backup n8n data directory
echo "Backing up n8n data..."
tar -czf "$BACKUP_DIR/n8n_data_backup_$TIMESTAMP.tar.gz" \
  -C ~/Apps/n8n data/

echo "Data backup completed: $BACKUP_DIR/n8n_data_backup_$TIMESTAMP.tar.gz"

# Remove backups older than RETENTION_DAYS
echo "Cleaning up old backups (older than $RETENTION_DAYS days)..."
find "$BACKUP_DIR" -name "n8n_data_backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup retention policy applied."
EOF

# Make script executable
chmod +x ~/Apps/n8n/backup_data.sh

# Test the script
~/Apps/n8n/backup_data.sh
```

Schedule automated data backups with cron (daily at 3:00 AM):

```bash
# Edit crontab
crontab -e

# Add the following line:
# 0 3 * * * ~/Apps/n8n/backup_data.sh

# Verify cron entry
crontab -l
```

#### Combined Backup Script

Create a comprehensive backup script:

```bash
cat > ~/Apps/n8n/backup_all.sh << 'EOF'
#!/bin/bash

set -e

BACKUP_DIR=~/Apps/n8n/backups
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

echo "=========================================="
echo "Starting n8n comprehensive backup"
echo "=========================================="
echo "Timestamp: $TIMESTAMP"

# Backup PostgreSQL database
echo "Backing up PostgreSQL database..."
docker exec n8n-postgres pg_dump -U n8n -d n8n | gzip > "$BACKUP_DIR/n8n_db_backup_$TIMESTAMP.sql.gz"
echo "✓ Database backup complete"

# Backup n8n data directory
echo "Backing up n8n data directory..."
tar -czf "$BACKUP_DIR/n8n_data_backup_$TIMESTAMP.tar.gz" \
  -C ~/Apps/n8n data/
echo "✓ Data backup complete"

# Create backup manifest
cat > "$BACKUP_DIR/n8n_backup_manifest_$TIMESTAMP.txt" << MANIFEST
Backup Timestamp: $TIMESTAMP
Database Backup: n8n_db_backup_$TIMESTAMP.sql.gz
Data Backup: n8n_data_backup_$TIMESTAMP.tar.gz

Backup Contents:
- PostgreSQL database (n8n database)
- n8n workflows, credentials, settings

Restore Instructions:
See RESTORE section in N8N_DEPLOYMENT_GUIDE.md
MANIFEST

echo "✓ Manifest created"

# Remove backups older than RETENTION_DAYS
echo "Cleaning up old backups (older than $RETENTION_DAYS days)..."
find "$BACKUP_DIR" -name "n8n_db_backup_*.sql.gz" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "n8n_data_backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "n8n_backup_manifest_*.txt" -mtime +$RETENTION_DAYS -delete

echo "✓ Cleanup complete"

echo "=========================================="
echo "Backup completed successfully"
echo "Location: $BACKUP_DIR"
ls -lh "$BACKUP_DIR" | tail -5
echo "=========================================="
EOF

# Make script executable
chmod +x ~/Apps/n8n/backup_all.sh

# Test the script
~/Apps/n8n/backup_all.sh
```

Schedule comprehensive backups (daily at 2:30 AM):

```bash
# Edit crontab
crontab -e

# Add the following line:
# 30 2 * * * ~/Apps/n8n/backup_all.sh >> ~/Apps/n8n/backups/backup_cron.log 2>&1

# Verify cron entry
crontab -l
```

### Database Restore

#### Full Database Restore

Restore PostgreSQL database from backup:

```bash
# Identify the backup file
ls -lh ~/Apps/n8n/backups/n8n_db_backup_*.sql.gz

# Before restoring, stop n8n to release database connections
docker stop n8n

# Restore database from compressed backup
zcat ~/Apps/n8n/backups/n8n_db_backup_YYYYMMDD_HHMMSS.sql.gz | \
  docker exec -i n8n-postgres psql -U n8n -d n8n

# Restart n8n
docker start n8n

# Verify restoration
docker logs n8n --tail=20 | grep -i "connected\|restored"
```

#### Drop and Restore Database

If the database is corrupted, drop and restore:

```bash
# Stop n8n
docker stop n8n

# Connect to PostgreSQL and drop the database
docker exec -i n8n-postgres psql -U n8n -c "DROP DATABASE n8n;"

# Recreate the database
docker exec -i n8n-postgres psql -U n8n -c "CREATE DATABASE n8n;"

# Restore from backup
zcat ~/Apps/n8n/backups/n8n_db_backup_YYYYMMDD_HHMMSS.sql.gz | \
  docker exec -i n8n-postgres psql -U n8n -d n8n

# Start n8n
docker start n8n

# Verify
docker logs n8n --tail=20
```

### Data Restore

#### Restore n8n Data Directory

Restore workflows, credentials, and settings from backup:

```bash
# Stop n8n container
docker stop n8n

# Backup current data (safety precaution)
tar -czf ~/Apps/n8n/backups/n8n_data_before_restore_$(date +%Y%m%d_%H%M%S).tar.gz \
  -C ~/Apps/n8n data/

# Remove current data directory
rm -rf ~/Apps/n8n/data/*

# Extract backup
tar -xzf ~/Apps/n8n/backups/n8n_data_backup_YYYYMMDD_HHMMSS.tar.gz \
  -C ~/Apps/n8n

# Start n8n
docker start n8n

# Verify
docker logs n8n --tail=20
```

### Workflow Export/Import

#### Export All Workflows

Export workflows from n8n using the CLI:

```bash
# Export all workflows
docker exec n8n n8n export:workflows --output=~/Apps/n8n/backups/n8n_workflows_$(date +%Y%m%d_%H%M%S).json

# Verify export
ls -lh ~/Apps/n8n/backups/n8n_workflows_*.json
```

#### Import Workflows

Import workflows into a new or existing n8n installation:

```bash
# Import workflows from backup
docker exec -i n8n n8n import:workflows < ~/Apps/n8n/backups/n8n_workflows_YYYYMMDD_HHMMSS.json

# Verify import
docker logs n8n --tail=20
```

---

## Updates & Upgrades

### Safe Update Process

Always perform backups before updating:

```bash
# Perform complete backup
~/Apps/n8n/backup_all.sh

# Verify backup completed
ls -lh ~/Apps/n8n/backups/ | tail -3
```

### Update n8n to Latest Stable Version

1. **Determine desired version:** Check [Docker Hub - n8n Tags](https://hub.docker.com/r/n8nio/n8n/tags)

2. **Update docker-compose.yml:**

```bash
# Edit docker-compose.yml
nano ~/Apps/n8n/docker-compose.yml

# Find the line: image: n8nio/n8n:YOUR_N8N_VERSION
# Replace YOUR_N8N_VERSION with the new version (e.g., 1.108.0)
```

3. **Update n8n container:**

```bash
# Navigate to application directory
cd ~/Apps/n8n

# Pull the new image
docker compose pull

# Expected output: Pulling n8n ... Downloaded newer image

# Restart container
docker compose up -d

# Wait 30-60 seconds for startup
sleep 45

# Verify successful update
docker logs n8n --tail=50 | grep -i "ready\|started\|error"
```

4. **Verify update:**

```bash
# Check n8n version
docker exec n8n n8n --version

# Access n8n editor and verify workflows are intact
# Browser: https://YOUR_N8N_DOMAIN
```

### Rollback to Previous Version

If an update causes issues, rollback to the previous version:

```bash
# Stop containers
docker compose down

# Edit docker-compose.yml to use the previous version
nano ~/Apps/n8n/docker-compose.yml

# Restart with previous version
docker compose up -d

# Wait for startup
sleep 45

# Verify
docker logs n8n --tail=50
```

### Update Ubuntu System

Periodically update OS and system packages:

```bash
# Update package manager
sudo apt update

# Check available updates
sudo apt list --upgradable

# Install updates (safe)
sudo apt upgrade -y

# Install major updates (use with caution)
# sudo apt full-upgrade -y

# Restart if kernel updated
sudo systemctl reboot
```

### Update Docker

Update Docker Engine to the latest version:

```bash
# Check current version
docker --version

# Update Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh

# Verify update
docker --version
```

---

## Migration to Another VM

This section explains how to migrate n8n from one VM to another while preserving all data, workflows, and credentials.

### Pre-Migration Checklist

Before starting migration:

- [ ] New VM created and accessible
- [ ] Ubuntu OS installed on new VM
- [ ] New VM has public IP address and DNS configured
- [ ] SSH key or password credentials for new VM
- [ ] Backup of current n8n installation complete
- [ ] Current domain/IP documented
- [ ] New domain/IP documented (if changing)

### Step 1: Complete Backup on Source VM

On the source (current) VM, create a complete backup:

```bash
# SSH to source VM
ssh YOUR_LINUX_USER@YOUR_PUBLIC_IP

# Create comprehensive backup
~/Apps/n8n/backup_all.sh

# List backup files
ls -lh ~/Apps/n8n/backups/

# Expected: Multiple recent backup files
```

### Step 2: Copy n8n Directory to New VM

Transfer the entire `~/Apps/n8n` directory to the new VM:

**Option A: Using rsync (recommended)**

```bash
# From your local machine, copy to new VM
rsync -avz --progress ~/Apps/n8n/ YOUR_LINUX_USER@NEW_IP:~/Apps/n8n/

# Expected: File transfer progress displayed
```

**Option B: Using scp**

```bash
# From the source VM, copy to new VM
scp -r ~/Apps/n8n YOUR_LINUX_USER@NEW_IP:~/Apps/
```

**Option C: Using tar via SSH**

```bash
# From the source VM, pipe to new VM
tar -czf - -C ~/Apps n8n | ssh YOUR_LINUX_USER@NEW_IP 'tar -xzf - -C ~/Apps'
```

### Step 3: Update Configuration on New VM

On the new VM, update configuration values that change:

```bash
# SSH to new VM
ssh YOUR_LINUX_USER@NEW_IP

# Edit .env file
nano ~/Apps/n8n/.env

# Update the following values:
# N8N_HOST=NEW_N8N_DOMAIN
# WEBHOOK_URL=https://NEW_N8N_DOMAIN/
# N8N_EDITOR_BASE_URL=https://NEW_N8N_DOMAIN/
# GENERIC_TIMEZONE=NEW_TIMEZONE (if different)

# Keep these values IDENTICAL to source VM:
# POSTGRES_PASSWORD
# N8N_ENCRYPTION_KEY

# Save and exit (Ctrl+X, then Y, then Enter)
```

### Step 4: Install Docker on New VM

On the new VM, install Docker and Docker Compose:

```bash
# Update OS
sudo apt update
sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

### Step 5: Verify Directory Structure

Verify the migrated directory structure on new VM:

```bash
# Check directory structure
ls -la ~/Apps/n8n/

# Expected output:
# drwxr-xr-x  data/
# drwxr-xr-x  postgres/
# drwxr-xr-x  backups/
# -rw-------  .env
# -rw-r--r--  docker-compose.yml
# -rwxr-xr-x  backup_*.sh scripts

# Verify .env file permissions
ls -la ~/Apps/n8n/.env

# Expected: -rw------- (600 permissions)
```

### Step 6: Start Containers on New VM

Start the migrated containers:

```bash
# Navigate to application directory
cd ~/Apps/n8n

# Start containers
docker compose up -d

# Wait for startup (45-60 seconds)
sleep 60

# Check container status
docker compose ps

# Expected: Both n8n and n8n-postgres running and healthy
```

### Step 7: Verify Data Integrity

Verify that all workflows and credentials migrated successfully:

```bash
# Check n8n logs
docker logs n8n --tail=50 | grep -i "ready\|error"

# Check database connectivity
docker exec n8n-postgres psql -U n8n -d n8n -c "SELECT COUNT(*) FROM workflow;"

# Access n8n editor
# Browser: https://NEW_N8N_DOMAIN

# Verify workflows are present
# Expected: All workflows from source VM visible in editor
```

### Step 8: Update DNS Records

Update DNS to point to the new VM's IP address:

```bash
# From your DNS provider:
# 1. Update A record for YOUR_N8N_DOMAIN
# 2. Point to NEW_PUBLIC_IP instead of OLD_PUBLIC_IP
# 3. Set TTL to 300 (5 minutes)
# 4. Wait for propagation (5-300 seconds)

# Verify DNS update from new VM
nslookup YOUR_N8N_DOMAIN 8.8.8.8

# Expected: NEW_PUBLIC_IP
```

### Step 9: Configure HTTPS on New VM

Set up Nginx and Let's Encrypt certificate on new VM:

```bash
# Follow the HTTPS Configuration section starting at:
# "Step 1: Install Nginx"

# Key steps:
# 1. Install Nginx
# 2. Create reverse proxy configuration
# 3. Install Certbot
# 4. Issue certificate with Let's Encrypt
```

### Step 10: Decommission Source VM

After verifying successful migration on the new VM:

```bash
# On source VM, save any additional data if needed
# Then optionally stop containers

docker compose down

# Backup source VM data (if desired)
tar -czf ~/n8n_source_backup_final_$(date +%Y%m%d).tar.gz ~/Apps/n8n/

# Terminate/delete source VM through cloud provider console
```

### Migration Verification Checklist

After migration, verify the following:

- [ ] n8n editor accessible at https://YOUR_N8N_DOMAIN
- [ ] Owner account logs in successfully
- [ ] All workflows present and runnable
- [ ] All credentials intact and functional
- [ ] Webhooks operational with new domain
- [ ] Database backups working on new VM
- [ ] DNS resolves to new IP
- [ ] SSL certificate valid and auto-renewal enabled
- [ ] Nginx logs show no errors
- [ ] Container health checks passing

---

## Troubleshooting

### Common Issues and Solutions

#### Issue: Docker Containers Won't Start

**Symptoms:**
- `docker compose up -d` fails
- Containers immediately exit or restart loops

**Diagnosis:**

```bash
# Check container logs
docker logs n8n
docker logs n8n-postgres

# Check Docker daemon status
sudo systemctl status docker

# Check system resources
free -h
df -h
```

**Solutions:**

1. **Out of disk space:**
   ```bash
   # Check available disk space
   df -h
   
   # Clean Docker system
   docker system prune -a
   ```

2. **Insufficient memory:**
   ```bash
   # Check memory
   free -h
   
   # Allocate more RAM to VM or reduce memory usage
   ```

3. **Port already in use:**
   ```bash
   # Check if port 5678 is in use
   sudo netstat -tulpn | grep 5678
   
   # If port is in use, kill the process
   sudo kill -9 <PID>
   ```

#### Issue: PostgreSQL Connection Failed

**Symptoms:**
- n8n logs show "Cannot connect to database"
- "password authentication failed"
- "n8n-postgres unhealthy"

**Diagnosis:**

```bash
# Check PostgreSQL health
docker compose ps

# Check PostgreSQL logs
docker logs n8n-postgres --tail=50

# Test PostgreSQL connectivity
docker exec n8n-postgres pg_isready -U n8n -d n8n
```

**Solutions:**

1. **Wrong password in .env file:**
   ```bash
   # Verify .env file
   cat ~/Apps/n8n/.env | grep POSTGRES_PASSWORD
   
   # Ensure POSTGRES_PASSWORD matches database password
   ```

2. **PostgreSQL initialization failed:**
   ```bash
   # Stop containers
   docker compose down
   
   # Remove postgres directory (data will be lost!)
   rm -rf ~/Apps/n8n/postgres
   mkdir ~/Apps/n8n/postgres
   
   # Restart
   docker compose up -d
   ```

3. **Insufficient disk space for database:**
   ```bash
   # Check disk space
   df -h
   
   # Clean Docker volumes and images
   docker volume prune
   docker image prune -a
   ```

#### Issue: n8n Won't Reach "Ready" State

**Symptoms:**
- n8n container keeps restarting
- Logs show repeated startup attempts
- n8n never becomes healthy

**Diagnosis:**

```bash
# Monitor startup logs
docker logs n8n -f

# Check for specific errors
docker logs n8n --tail=100 | grep -i "error\|failed\|exception"
```

**Solutions:**

1. **Encryption key mismatch:**
   ```bash
   # If migrating or restoring, encryption key must match
   # Check .env file
   cat ~/Apps/n8n/.env | grep ENCRYPTION_KEY
   
   # If key is wrong, update .env and restart
   docker compose restart n8n
   ```

2. **Missing encryption key:**
   ```bash
   # Verify N8N_ENCRYPTION_KEY is set in .env
   grep "N8N_ENCRYPTION_KEY" ~/Apps/n8n/.env
   
   # If missing, generate and add
   openssl rand -hex 32
   # Add to .env file
   nano ~/Apps/n8n/.env
   
   # Restart n8n
   docker compose restart n8n
   ```

3. **Config file corruption:**
   ```bash
   # Reset n8n config
   docker compose down
   rm -rf ~/Apps/n8n/data/config
   docker compose up -d
   ```

#### Issue: HTTPS Certificate Failed

**Symptoms:**
- "Your connection is not private" in browser
- Certbot fails to issue certificate
- Certificate expires without renewal

**Diagnosis:**

```bash
# Check certificate status
sudo certbot certificates

# Check certificate expiration
echo | openssl s_client -connect YOUR_N8N_DOMAIN:443 -servername YOUR_N8N_DOMAIN 2>/dev/null | openssl x509 -noout -dates

# Check Certbot logs
sudo tail -50 /var/log/letsencrypt/letsencrypt.log

# Test renewal
sudo certbot renew --dry-run --verbose
```

**Solutions:**

1. **DNS not resolving:**
   ```bash
   # Verify DNS resolution
   nslookup YOUR_N8N_DOMAIN 8.8.8.8
   
   # If not resolving, update DNS A record to YOUR_PUBLIC_IP
   # Wait for propagation (up to 300 seconds)
   ```

2. **Port 80 not accessible:**
   ```bash
   # Check firewall allows port 80
   sudo ufw status
   
   # If not allowed, add rule
   sudo ufw allow 80/tcp
   
   # Check cloud provider security group allows port 80
   ```

3. **Certificate renewal disabled:**
   ```bash
   # Enable and start Certbot renewal timer
   sudo systemctl enable certbot.timer
   sudo systemctl start certbot.timer
   
   # Check status
   sudo systemctl status certbot.timer
   ```

#### Issue: DNS Not Resolving

**Symptoms:**
- Browser shows "Cannot connect to server"
- `nslookup YOUR_N8N_DOMAIN` returns "server not found"
- Nginx returns "Connection refused"

**Diagnosis:**

```bash
# Test DNS resolution from different DNS servers
nslookup YOUR_N8N_DOMAIN        # Default resolver
nslookup YOUR_N8N_DOMAIN 8.8.8.8 # Google DNS
dig YOUR_N8N_DOMAIN              # Detailed DNS info

# Check if domain is registered
whois YOUR_N8N_DOMAIN

# Test connectivity to IP
ping YOUR_PUBLIC_IP
curl http://YOUR_PUBLIC_IP
```

**Solutions:**

1. **DNS A record not created or incorrect:**
   ```bash
   # Verify A record in DNS provider:
   # Hostname: n8n (or your prefix)
   # Type: A
   # Value: YOUR_PUBLIC_IP
   # TTL: 300
   
   # Wait for propagation: 5-300 seconds
   ```

2. **TTL too high:**
   ```bash
   # Lower TTL in DNS provider to 300 (5 minutes)
   # This allows faster DNS updates
   ```

3. **Cached DNS:**
   ```bash
   # Flush local DNS cache
   # macOS:
   sudo dscacheutil -flushcache
   
   # Windows (in Command Prompt):
   ipconfig /flushdns
   
   # Linux:
   sudo systemctl restart systemd-resolved
   ```

#### Issue: 502 Bad Gateway

**Symptoms:**
- Browser shows "502 Bad Gateway" error
- Nginx logs show upstream errors
- n8n container is running but not responding

**Diagnosis:**

```bash
# Check Nginx error logs
sudo tail -50 /var/log/nginx/error.log

# Check if n8n is listening
docker exec n8n lsof -i :5678

# Test local connectivity to n8n
curl http://127.0.0.1:5678

# Check n8n container status
docker ps | grep n8n
```

**Solutions:**

1. **n8n not responding:**
   ```bash
   # Restart n8n container
   docker restart n8n
   
   # Wait 30 seconds for startup
   sleep 30
   
   # Check logs
   docker logs n8n --tail=50
   ```

2. **n8n container unhealthy:**
   ```bash
   # Check container status
   docker compose ps
   
   # If unhealthy, check database connection
   docker logs n8n --tail=50 | grep -i "postgres\|database"
   ```

3. **Nginx reverse proxy misconfigured:**
   ```bash
   # Check Nginx configuration
   sudo nginx -t
   
   # If error, review config
   sudo nano /etc/nginx/sites-available/default
   
   # Reload Nginx after fixing
   sudo systemctl reload nginx
   ```

#### Issue: Cannot Access Owner Account

**Symptoms:**
- Owner account creation page not appearing
- "Owner account already exists" message
- Login fails with correct credentials

**Diagnosis:**

```bash
# Check if data folder exists
ls -la ~/Apps/n8n/data/

# Query database for users
docker exec n8n-postgres psql -U n8n -d n8n -c "SELECT * FROM user;"

# Check n8n config
cat ~/Apps/n8n/data/config
```

**Solutions:**

1. **Data folder exists but owner account not created:**
   ```bash
   # Access n8n at https://YOUR_N8N_DOMAIN
   # If setup page appears, complete owner account creation
   ```

2. **Need to reset owner account:**
   ```bash
   # WARNING: This deletes all data!
   docker compose down
   rm -rf ~/Apps/n8n/data
   mkdir ~/Apps/n8n/data
   docker compose up -d
   
   # Access https://YOUR_N8N_DOMAIN to create new owner account
   ```

3. **Forgot owner password:**
   ```bash
   # n8n does not provide password reset via UI
   # Options:
   # 1. Reset database (see above)
   # 2. Use database to update password (advanced)
   ```

#### Issue: Workflows Running Slowly

**Symptoms:**
- Workflow execution takes longer than expected
- n8n UI is sluggish or unresponsive
- WebSocket connections timing out

**Diagnosis:**

```bash
# Check system resources
docker stats n8n n8n-postgres

# Check database performance
docker exec n8n-postgres psql -U n8n -d n8n -c "SELECT COUNT(*) FROM workflow_statistics;"

# Check n8n logs for slowness
docker logs n8n --tail=100 | grep -i "slow\|timeout\|took"

# Check Nginx timeout settings
sudo cat /etc/nginx/sites-available/default | grep timeout
```

**Solutions:**

1. **Insufficient VM resources:**
   ```bash
   # Check available memory and CPU
   free -h
   top -b -n 1
   
   # Upgrade VM resources if consistently full
   ```

2. **Database performance issues:**
   ```bash
   # Vacuum PostgreSQL database
   docker exec n8n-postgres vacuumdb -U n8n -d n8n -v
   
   # Analyze database
   docker exec n8n-postgres analyzedb -U n8n -d n8n
   ```

3. **WebSocket timeout:**
   ```bash
   # Increase Nginx proxy timeout
   sudo nano /etc/nginx/sites-available/default
   
   # Increase these values:
   # proxy_read_timeout 3600;
   # proxy_send_timeout 3600;
   # proxy_connect_timeout 3600;
   
   # Reload Nginx
   sudo systemctl reload nginx
   ```

#### Issue: Webhooks Not Working

**Symptoms:**
- Workflows triggered by webhooks don't execute
- "Webhook not found" errors
- External systems can't reach webhook URL

**Diagnosis:**

```bash
# Test webhook URL from local machine
curl -X POST https://YOUR_N8N_DOMAIN/webhook/YOUR_WORKFLOW_ID

# Check webhook configuration in n8n
# Open n8n editor and inspect webhook node

# Check Nginx logs for webhook requests
sudo tail -50 /var/log/nginx/access.log | grep webhook

# Check Nginx error logs
sudo tail -50 /var/log/nginx/error.log
```

**Solutions:**

1. **Domain changed but webhook URL not updated:**
   ```bash
   # In n8n editor:
   # 1. Edit workflow with webhook
   # 2. Webhook node shows new URL with updated domain
   # 3. Update webhook URL in external systems
   ```

2. **WEBHOOK_URL environment variable not set correctly:**
   ```bash
   # Check .env file
   cat ~/Apps/n8n/.env | grep WEBHOOK_URL
   
   # Should be: https://YOUR_N8N_DOMAIN/
   # Update if necessary
   nano ~/Apps/n8n/.env
   
   # Restart n8n
   docker compose restart n8n
   ```

3. **Firewall blocking webhook requests:**
   ```bash
   # Verify ports 80 and 443 are open
   sudo ufw status
   
   # Verify cloud provider security group allows these ports
   # Update if necessary in cloud provider console
   ```

---

## Security Best Practices

### SSH Key-Based Authentication

Disable password authentication and use SSH keys only:

```bash
# Generate SSH key pair (on your local machine)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Add public key to authorized_keys on VM
cat ~/.ssh/id_ed25519.pub | ssh YOUR_LINUX_USER@YOUR_PUBLIC_IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Set correct permissions
ssh YOUR_LINUX_USER@YOUR_PUBLIC_IP "chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"

# Test SSH key authentication
ssh -i ~/.ssh/id_ed25519 YOUR_LINUX_USER@YOUR_PUBLIC_IP

# Disable password authentication (on VM)
sudo nano /etc/ssh/sshd_config

# Find and change:
# PasswordAuthentication yes  →  PasswordAuthentication no
# PubkeyAuthentication no     →  PubkeyAuthentication yes

# Restart SSH service
sudo systemctl restart sshd

# Verify SSH key works before logging out
```

### Environment File Security

Protect the `.env` file containing sensitive data:

```bash
# Set restrictive permissions (readable only by owner)
chmod 600 ~/Apps/n8n/.env

# Verify permissions
ls -la ~/Apps/n8n/.env

# Expected: -rw------- (600)

# Ensure .env is not in version control
echo ".env" >> ~/.gitignore
```

### Secrets Management

For production deployments, use a secrets manager instead of `.env` files:

**Options:**
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- 1Password
- Bitwarden

These provide:
- Encrypted storage
- Audit trails
- Access control
- Automatic rotation
- API integration with Docker

### Regular Updates

Keep all software up to date:

```bash
# Ubuntu system updates
sudo apt update
sudo apt upgrade -y

# Docker engine updates
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Docker image updates
cd ~/Apps/n8n
docker compose pull
docker compose up -d

# n8n version pinning
# Always update to specific known stable versions
# Never use 'latest' tag in production
```

### Firewall Hardening

Restrict network access:

```bash
# Limit SSH to specific IP ranges
sudo ufw allow from YOUR_IP/32 to any port 22/tcp

# Close unnecessary ports
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow only required ports
sudo ufw allow 22/tcp  # SSH (restricted source)
sudo ufw allow 80/tcp  # HTTP (Let's Encrypt)
sudo ufw allow 443/tcp # HTTPS

# Do NOT allow port 5678 (n8n internal)

# Enable UFW
sudo ufw enable

# Verify rules
sudo ufw status
```

### Database Security

Protect PostgreSQL database:

```bash
# Use strong password (minimum 24 characters)
echo $YOUR_POSTGRES_PASSWORD | wc -c

# Expected: 25+ (includes newline)

# Limit database access to n8n container only
# (Already done in docker-compose.yml with internal network)

# Regular backups
~/Apps/n8n/backup_all.sh

# Database encryption at rest
# PostgreSQL does not provide built-in encryption
# Use full-disk encryption or encrypted cloud storage
```

### Nginx Security Headers

Add security headers to Nginx:

```bash
# Edit Nginx configuration
sudo nano /etc/nginx/sites-available/default

# Add to the server block:
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;

# Reload Nginx
sudo nginx -t
sudo systemctl reload nginx
```

### Monitoring & Logging

Enable comprehensive logging:

```bash
# Set n8n log level
# In .env: N8N_LOG_LEVEL=info

# Monitor logs in real-time
docker logs n8n -f

# Archive logs for analysis
docker logs n8n > ~/Apps/n8n/backups/n8n_logs_$(date +%Y%m%d).txt

# Monitor system resources
docker stats n8n n8n-postgres

# Enable audit logging
# See Nginx access logs
sudo tail -f /var/log/nginx/access.log
```

### Backup Encryption

Encrypt backups before storing:

```bash
# Create backup and encrypt
~/Apps/n8n/backup_all.sh

# Encrypt backup
cd ~/Apps/n8n/backups
gpg --symmetric n8n_db_backup_YYYYMMDD_HHMMSS.sql.gz

# Expected: creates .gpg file with encryption password

# Decrypt backup
gpg --output n8n_db_backup_YYYYMMDD_HHMMSS.sql n8n_db_backup_YYYYMMDD_HHMMSS.sql.gz.gpg

# Remove unencrypted backup
rm n8n_db_backup_YYYYMMDD_HHMMSS.sql.gz
```

---

## Quick Reference

### Essential Commands

#### Container Management

```bash
# Start services
cd ~/Apps/n8n && docker compose up -d

# Stop services
docker compose down

# Restart services
docker compose restart

# View container status
docker compose ps

# View running containers
docker ps
```

#### Logging

```bash
# View n8n logs (last 50 lines)
docker logs n8n --tail=50

# View n8n logs in real-time
docker logs n8n -f

# View PostgreSQL logs
docker logs n8n-postgres --tail=50

# View Nginx error logs
sudo tail -50 /var/log/nginx/error.log

# View Nginx access logs
sudo tail -50 /var/log/nginx/access.log
```

#### Database Management

```bash
# Access PostgreSQL shell
docker exec -it n8n-postgres psql -U n8n -d n8n

# Backup database
docker exec n8n-postgres pg_dump -U n8n -d n8n | gzip > ~/Apps/n8n/backups/backup_$(date +%Y%m%d_%H%M%S).sql.gz

# Restore database
zcat ~/Apps/n8n/backups/backup.sql.gz | docker exec -i n8n-postgres psql -U n8n -d n8n

# View database size
docker exec n8n-postgres psql -U n8n -d n8n -c "SELECT pg_size_pretty(pg_database_size('n8n'));"
```

#### Backup & Restore

```bash
# Backup everything
~/Apps/n8n/backup_all.sh

# Restore database
# See Backup & Restore section

# Restore data directory
# See Backup & Restore section

# Export workflows
docker exec n8n n8n export:workflows --output=~/Apps/n8n/backups/workflows_$(date +%Y%m%d).json

# Import workflows
docker exec -i n8n n8n import:workflows < ~/Apps/n8n/backups/workflows.json
```

#### Updates & Maintenance

```bash
# Update n8n image
docker compose pull && docker compose up -d

# Update Ubuntu
sudo apt update && sudo apt upgrade -y

# Check certificate expiration
echo | openssl s_client -connect YOUR_N8N_DOMAIN:443 2>/dev/null | openssl x509 -noout -dates

# Check Certbot renewal status
sudo certbot certificates
```

#### System Information

```bash
# View container resource usage
docker stats n8n n8n-postgres

# View system resources
free -h
df -h
top -b -n 1

# View n8n version
docker exec n8n n8n --version

# View PostgreSQL version
docker exec n8n-postgres psql --version
```

### Directory Structure

Complete directory layout for production deployment:

```
~/Apps/n8n/
├── docker-compose.yml          # Docker Compose service definitions
├── .env                        # Environment variables (keep secret!)
│
├── data/                       # n8n data volume
│   ├── workflows/              # Stored workflows
│   ├── credentials.json        # Encrypted credentials
│   └── config                  # n8n configuration
│
├── postgres/                   # PostgreSQL data volume
│   ├── base/
│   ├── global/
│   └── pg_wal/
│
├── backups/                    # Backup directory
│   ├── n8n_db_backup_*.sql.gz
│   ├── n8n_data_backup_*.tar.gz
│   ├── n8n_workflows_*.json
│   └── n8n_backup_manifest_*.txt
│
├── backup_all.sh               # Comprehensive backup script
├── backup_database.sh          # Database-only backup script
├── backup_data.sh              # Data-only backup script
├── DEPLOYMENT_INFO.txt         # Deployment record
└── DEPLOYMENT_GUIDE.md         # This file
```

### Important Paths

| Path | Purpose |
|------|---------|
| `~/Apps/n8n` | Main application directory |
| `~/Apps/n8n/data` | n8n workflows and credentials |
| `~/Apps/n8n/postgres` | PostgreSQL database storage |
| `~/Apps/n8n/backups` | Automated backups |
| `~/Apps/n8n/.env` | Configuration (KEEP SECRET) |
| `/etc/nginx/sites-available/default` | Nginx reverse proxy config |
| `/etc/letsencrypt/live/YOUR_N8N_DOMAIN/` | SSL certificates |
| `/var/log/nginx/` | Nginx logs |
| `/var/lib/docker/volumes/` | Docker volumes |

### Common Placeholder Values Summary

When executing commands from this guide, replace:

| Replace This | With This |
|-------------|-----------|
| `YOUR_N8N_DOMAIN` | Your n8n domain (e.g., `n8n.example.com`) |
| `YOUR_PUBLIC_IP` | Your VM's public IP address |
| `YOUR_LINUX_USER` | Your Linux username (e.g., `ubuntu`) |
| `YOUR_POSTGRES_PASSWORD` | Your PostgreSQL password (24+ chars) |
| `YOUR_N8N_ENCRYPTION_KEY` | Your n8n encryption key (64 hex chars) |
| `YOUR_TIMEZONE` | Your server timezone (e.g., `UTC`) |
| `YOUR_EMAIL` | Your email for certificates |
| `YOUR_N8N_VERSION` | Specific n8n version (e.g., `1.107.4`) |

---

## Status and Support

**Document Status:** Production-Ready  
**Last Updated:** 2026-07-13  
**n8n Documentation:** https://docs.n8n.io  
**n8n GitHub Issues:** https://github.com/n8n-io/n8n/issues  
**n8n Community Forum:** https://community.n8n.io

### Getting Help

If issues persist after reviewing the troubleshooting section:

1. Check n8n logs for error messages: `docker logs n8n --tail=100`
2. Review the Troubleshooting section in this guide
3. Search [n8n GitHub Issues](https://github.com/n8n-io/n8n/issues)
4. Ask in [n8n Community Forum](https://community.n8n.io)
5. Contact your cloud provider support for infrastructure issues

---

**End of N8N Deployment Guide**
