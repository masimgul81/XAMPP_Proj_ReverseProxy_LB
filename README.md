# XAMPP Docker Stack with Nginx Reverse Proxy & Load Balancing

A fully containerized XAMPP web server stack with Nginx reverse proxy, SSL termination, virtual hosts, and load balancing across multiple Apache instances.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Prerequisites & Installation](#prerequisites--installation)
3. [Project Structure](#project-structure)
4. [Configuration Files](#configuration-files)
5. [Step-by-Step Setup](#step-by-step-setup)
6. [SSL Certificate Setup](#ssl-certificate-setup)
7. [MySQL Database Setup](#mysql-database-setup)
8. [Load Balancing](#load-balancing)
9. [Testing Devices](#testing-devices)
10. [Test Criteria & Results](#test-criteria--results)
11. [Useful Commands](#useful-commands)
12. [Troubleshooting](#troubleshooting)

---

## Architecture

```
Windows Browser (HTTPS — no port needed)
            │
            ▼
    ┌───────────────┐
    │  Nginx :443   │  ← SSL Termination (mkcert trusted certs)
    │  Nginx :80    │  ← HTTP → HTTPS redirect (301)
    └───────┬───────┘
            │  least_conn load balancing
            │
    ┌───────┼───────────────────┐
    ▼       ▼                   ▼
xampp1:8081/8082/8083   xampp2:8081/8082/8083   xampp3:8081/8082/8083
(Apache + MySQL)         (Apache only)           (Apache only)
    │
    ▼
MySQL :3306
(shared volume — xampp1 only)
```

### Port Mapping Summary

| Port | Service | Accessible From |
|------|---------|----------------|
| 80 | Nginx HTTP | External (redirects to 443) |
| 443 | Nginx HTTPS | External (SSL termination) |
| 3306 | MySQL | External (direct DB access) |
| 8081–8083 | XAMPP Apache | Internal only (Nginx → XAMPP) |

---

## Prerequisites & Installation

### Server Requirements

| Component | Version |
|-----------|---------|
| Ubuntu Server | 22.04 LTS or later |
| Docker Engine | 24.x or later |
| Docker Compose | v2 plugin or v1 standalone |

### Install Docker on Ubuntu

```bash
# Update package index
sudo apt update

# Install Docker
sudo apt install -y docker.io

# Install Docker Compose plugin (v2)
sudo apt install -y docker-compose-plugin

# Add your user to docker group (avoid using sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

### Install Docker Compose v1 (alternative)

```bash
sudo apt install -y docker-compose
docker-compose --version
```

### Install mkcert on Windows (for trusted SSL certs)

Via Winget:
```powershell
winget install FiloSottile.mkcert
```

Via Chocolatey:
```powershell
choco install mkcert
```

Or download manually from:
`https://github.com/FiloSottile/mkcert/releases/latest`
→ Download `mkcert-v*-windows-amd64.exe`, rename to `mkcert.exe`, place in `C:\Windows\System32\`

---

## Project Structure

```
xampp/
├── docker-compose.yml
├── config/
│   ├── httpd.conf                  # Apache main config
│   ├── httpd-vhosts.conf           # Virtual host definitions
│   └── ssl/
│       ├── site1.crt / site1.key
│       ├── site2.crt / site2.key
│       └── site3.crt / site3.key
├── nginx/
│   └── nginx.conf                  # Nginx reverse proxy + load balancer config
└── htdocs/
    ├── index.html                  # Default site (port 80)
    ├── site1/
    │   ├── index.html
    │   ├── which.php               # Load balancer test file
    │   └── config.php              # DB connection config
    ├── site2/
    │   ├── index.html
    │   └── config.php
    └── site3/
        ├── index.html
        └── config.php
```

---

## Configuration Files

### docker-compose.yml

```yaml
version: "3.9"

services:

  # XAMPP Instance 1 (Apache + MySQL)
  xampp1:
    image: tomsik68/xampp:latest
    container_name: xampp1
    hostname: xampp1
    expose:
      - "8081"
      - "8082"
      - "8083"
    ports:
      - "3306:3306"
    volumes:
      - ./htdocs:/www
      - xampp_data:/opt/lampp/var/mysql
      - ./config/httpd.conf:/opt/lampp/etc/httpd.conf:ro
      - ./config/httpd-vhosts.conf:/opt/lampp/etc/extra/httpd-vhosts.conf:ro
    networks:
      - web
    restart: unless-stopped

  # XAMPP Instance 2 (Apache only)
  xampp2:
    image: tomsik68/xampp:latest
    container_name: xampp2
    hostname: xampp2
    expose:
      - "8081"
      - "8082"
      - "8083"
    volumes:
      - ./htdocs:/www
      - ./config/httpd.conf:/opt/lampp/etc/httpd.conf:ro
      - ./config/httpd-vhosts.conf:/opt/lampp/etc/extra/httpd-vhosts.conf:ro
    networks:
      - web
    restart: unless-stopped

  # XAMPP Instance 3 (Apache only)
  xampp3:
    image: tomsik68/xampp:latest
    container_name: xampp3
    hostname: xampp3
    expose:
      - "8081"
      - "8082"
      - "8083"
    volumes:
      - ./htdocs:/www
      - ./config/httpd.conf:/opt/lampp/etc/httpd.conf:ro
      - ./config/httpd-vhosts.conf:/opt/lampp/etc/extra/httpd-vhosts.conf:ro
    networks:
      - web
    restart: unless-stopped

  # Nginx Reverse Proxy + Load Balancer
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/ssl:/etc/nginx/ssl:ro
    depends_on:
      - xampp1
      - xampp2
      - xampp3
    networks:
      - web
    restart: unless-stopped

volumes:
  xampp_data:

networks:
  web:
    driver: bridge
```

---

### config/httpd.conf (key sections)

```apache
# Listen ports — one per virtual host
Listen 80
Listen 8081
Listen 8082
Listen 8083

# Default document root — must match volume mount
DocumentRoot "/www"
<Directory "/www">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>

# Enable virtual hosts
Include etc/extra/httpd-vhosts.conf
```

---

### config/httpd-vhosts.conf

```apache
# Default site
<VirtualHost *:80>
    DocumentRoot "/www"
    ServerName localhost
</VirtualHost>

# Site 1
<VirtualHost *:8081>
    DocumentRoot "/www/site1"
    ServerName site1.local
    <Directory "/www/site1">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# Site 2
<VirtualHost *:8082>
    DocumentRoot "/www/site2"
    ServerName site2.local
    <Directory "/www/site2">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# Site 3
<VirtualHost *:8083>
    DocumentRoot "/www/site3"
    ServerName site3.local
    <Directory "/www/site3">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

---

### nginx/nginx.conf

```nginx
events {}

http {

    # Load balancer upstream pools
    upstream site1_pool {
        least_conn;
        server xampp1:8081;
        server xampp2:8081;
        server xampp3:8081;
        keepalive 0;
    }

    upstream site2_pool {
        least_conn;
        server xampp1:8082;
        server xampp2:8082;
        server xampp3:8082;
        keepalive 0;
    }

    upstream site3_pool {
        least_conn;
        server xampp1:8083;
        server xampp2:8083;
        server xampp3:8083;
        keepalive 0;
    }

    # site1.local
    server {
        listen 80;
        server_name site1.local;
        return 301 https://$host$request_uri;
    }
    server {
        listen 443 ssl;
        server_name site1.local;
        ssl_certificate     /etc/nginx/ssl/site1.crt;
        ssl_certificate_key /etc/nginx/ssl/site1.key;
        location / {
            proxy_pass         http://site1_pool;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }

    # site2.local
    server {
        listen 80;
        server_name site2.local;
        return 301 https://$host$request_uri;
    }
    server {
        listen 443 ssl;
        server_name site2.local;
        ssl_certificate     /etc/nginx/ssl/site2.crt;
        ssl_certificate_key /etc/nginx/ssl/site2.key;
        location / {
            proxy_pass         http://site2_pool;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }

    # site3.local
    server {
        listen 80;
        server_name site3.local;
        return 301 https://$host$request_uri;
    }
    server {
        listen 443 ssl;
        server_name site3.local;
        ssl_certificate     /etc/nginx/ssl/site3.crt;
        ssl_certificate_key /etc/nginx/ssl/site3.key;
        location / {
            proxy_pass         http://site3_pool;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }
}
```

---

### htdocs/site1/which.php (load balancer test file)

```php
<?php
echo "Served by container: " . gethostname();
?>
```

### htdocs/site1/config.php (database connection)

```php
<?php
define('DB_HOST', 'localhost');
define('DB_NAME', 'site1_db');
define('DB_USER', 'site1_user');
define('DB_PASS', 'site1_pass');

$pdo = new PDO(
    "mysql:host=" . DB_HOST . ";dbname=" . DB_NAME,
    DB_USER,
    DB_PASS
);
```

---

## Step-by-Step Setup

### Step 1 — Clone or create the project directory

```bash
mkdir ~/xampp && cd ~/xampp
```

### Step 2 — Create directory structure

```bash
mkdir -p config/ssl nginx htdocs/site1 htdocs/site2 htdocs/site3
```

### Step 3 — Extract default Apache configs from container

```bash
# Start a temporary container to copy configs
docker run -d --name temp_xampp tomsik68/xampp:latest
docker cp temp_xampp:/opt/lampp/etc/httpd.conf ./config/httpd.conf
docker cp temp_xampp:/opt/lampp/etc/extra/httpd-vhosts.conf ./config/httpd-vhosts.conf
docker rm -f temp_xampp
```

### Step 4 — Edit configs

Edit `./config/httpd.conf`:
- Change `DocumentRoot "/opt/lampp/htdocs"` → `DocumentRoot "/www"`
- Change `<Directory "/opt/lampp/htdocs">` → `<Directory "/www">`
- Uncomment `Include etc/extra/httpd-vhosts.conf`
- Add `Listen 8081`, `Listen 8082`, `Listen 8083`

Replace `./config/httpd-vhosts.conf` with the virtual host config above.

### Step 5 — Create site content

```bash
echo "Default Site port 80"   > ./htdocs/index.html
echo "Default Site port 8081" > ./htdocs/site1/index.html
echo "Default Site port 8082" > ./htdocs/site2/index.html
echo "Default Site port 8083" > ./htdocs/site3/index.html
```

### Step 6 — Generate SSL certificates (on Windows)

```powershell
# Run as Administrator — install local CA once
mkcert -install

# Generate certs for all 3 sites
mkcert.exe site1.local site2.local site3.local

# Copy to Ubuntu server
scp site1.local+2.pem     ubuntu@<SERVER_IP>:~/xampp/config/ssl/site1.crt
scp site1.local+2.pem     ubuntu@<SERVER_IP>:~/xampp/config/ssl/site2.crt
scp site1.local+2.pem     ubuntu@<SERVER_IP>:~/xampp/config/ssl/site3.crt
scp site1.local+2-key.pem ubuntu@<SERVER_IP>:~/xampp/config/ssl/site1.key
scp site1.local+2-key.pem ubuntu@<SERVER_IP>:~/xampp/config/ssl/site2.key
scp site1.local+2-key.pem ubuntu@<SERVER_IP>:~/xampp/config/ssl/site3.key
```

### Step 7 — Add hosts entries on Windows

Open Notepad as Administrator → `C:\Windows\System32\drivers\etc\hosts`:

```
192.168.11.129   site1.local
192.168.11.129   site2.local
192.168.11.129   site3.local
```

### Step 8 — Start the stack

```bash
docker-compose up -d

# Verify all 4 containers are running
docker ps
```

### Step 9 — Create MySQL databases

```bash
docker exec -it xampp1 /opt/lampp/bin/mysql -u root
```

```sql
CREATE DATABASE site1_db;
CREATE USER 'site1_user'@'localhost' IDENTIFIED BY 'site1_pass';
GRANT ALL PRIVILEGES ON site1_db.* TO 'site1_user'@'localhost';

CREATE DATABASE site2_db;
CREATE USER 'site2_user'@'localhost' IDENTIFIED BY 'site2_pass';
GRANT ALL PRIVILEGES ON site2_db.* TO 'site2_user'@'localhost';

CREATE DATABASE site3_db;
CREATE USER 'site3_user'@'localhost' IDENTIFIED BY 'site3_pass';
GRANT ALL PRIVILEGES ON site3_db.* TO 'site3_user'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

---

## SSL Certificate Setup

### Why mkcert?

Self-signed certificates generate browser warnings. mkcert creates a local Certificate Authority (CA) that is trusted by Windows, Chrome, Firefox, and curl — eliminating all SSL warnings for local development.

### mkcert vs Self-Signed Comparison

| Feature | Self-Signed | mkcert |
|---------|------------|--------|
| Browser warnings | Yes | No |
| curl -k required | Yes | No |
| Setup complexity | Low | Medium |
| Trust scope | None | Local machine |
| Expiry | Configurable | 2 years |

### Certificate file locations

| File | Container path | Purpose |
|------|---------------|---------|
| site1.crt | /etc/nginx/ssl/site1.crt | Certificate (public) |
| site1.key | /etc/nginx/ssl/site1.key | Private key |

---

## MySQL Database Setup

### Database to Site Mapping

| VirtualHost | Port | Database | User |
|-------------|------|----------|------|
| Default | 80 | — | — |
| site1.local | 8081 | site1_db | site1_user |
| site2.local | 8082 | site2_db | site2_user |
| site3.local | 8083 | site3_db | site3_user |

### phpMyAdmin Access

```
http://localhost/phpmyadmin
```

MySQL is built into XAMPP — phpMyAdmin is available on the default site.

---

## Load Balancing

### Method: Least Connections (`least_conn`)

Nginx routes each new request to the backend with the fewest active connections. This is ideal when requests have variable processing times.

### Available Load Balancing Methods

| Method | Config | Use Case |
|--------|--------|---------|
| Round Robin | *(default)* | Equal traffic distribution |
| Least Connections | `least_conn` | Variable request durations |
| IP Hash | `ip_hash` | Session persistence |
| Weighted | `weight=N` | Unequal server capacity |

### Weighted Example

```nginx
upstream site1_pool {
    server xampp1:8081 weight=3;   # receives 60% of traffic
    server xampp2:8081 weight=1;   # receives 20% of traffic
    server xampp3:8081 weight=1;   # receives 20% of traffic
}
```

### Container Roles

| Container | Role | MySQL |
|-----------|------|-------|
| xampp1 | Apache + MySQL | Primary (runs MySQL) |
| xampp2 | Apache only | No (avoids volume conflict) |
| xampp3 | Apache only | No (avoids volume conflict) |

> **Note:** Only xampp1 runs MySQL. Running MySQL on all instances against the same data volume causes startup conflicts. xampp2 and xampp3 serve web content only.

---

## Testing Devices

| Device | OS | Browser | Role |
|--------|-----|---------|------|
| ansibleserver | Ubuntu 22.04 | curl | Server / CLI testing |
| Windows workstation | Windows 11 | Chrome (Incognito) | Browser testing |

### Network Details

| Component | IP / Hostname |
|-----------|--------------|
| Ubuntu Server | 192.168.11.129 |
| Server hostname | ansibleserver |
| Test domains | site1.local, site2.local, site3.local |

---

## Test Criteria & Results

### TC-01: Container Health

**Criteria:** All 4 containers start and remain running.

```bash
docker ps
```

**Expected:**
```
NAMES     STATUS
nginx     Up
xampp1    Up
xampp2    Up
xampp3    Up
```

**Result:** ✅ Pass

---

### TC-02: Virtual Host Routing

**Criteria:** Each port serves the correct site content.

```bash
curl -sk https://site1.local/index.html   # → "Default Site port 8081"
curl -sk https://site2.local/index.html   # → "Default Site port 8082"
curl -sk https://site3.local/index.html   # → "Default Site port 8083"
```

**Result:** ✅ Pass

---

### TC-03: HTTP to HTTPS Redirect

**Criteria:** HTTP requests are redirected to HTTPS with a 301 status.

```bash
curl -I http://site1.local
```

**Expected:** `HTTP/1.1 301 Moved Permanently`

**Result:** ✅ Pass

---

### TC-04: SSL Certificate Trust

**Criteria:** Browser shows no SSL warning (padlock visible).

**Test:** Open `https://site1.local` in Chrome on Windows.

**Expected:** Green padlock, no certificate warning.

**Result:** ✅ Pass — mkcert CA trusted by Windows certificate store.

---

### TC-05: Apache Config Validation

**Criteria:** Apache config syntax is valid on all instances.

```bash
docker exec xampp1 /opt/lampp/bin/httpd -t
docker exec xampp2 /opt/lampp/bin/httpd -t
docker exec xampp3 /opt/lampp/bin/httpd -t
```

**Expected:** `Syntax OK`

**Result:** ✅ Pass

---

### TC-06: Virtual Host Registration

**Criteria:** Apache registers all 4 virtual hosts correctly.

```bash
docker exec xampp1 /opt/lampp/bin/httpd -S
```

**Expected:**
```
*:80    localhost
*:8081  site1.local
*:8082  site2.local
*:8083  site3.local
```

**Result:** ✅ Pass

---

### TC-07: Load Balancer Distribution

**Criteria:** Requests are distributed across all 3 XAMPP instances.

```bash
# From Ubuntu
for i in 1 2 3 4 5 6; do curl -sk https://site1.local/which.php; echo; done
```

**Expected:** Responses rotate between xampp1, xampp2, xampp3.

```
Served by container: xampp1
Served by container: xampp2
Served by container: xampp3
Served by container: xampp1
Served by container: xampp2
Served by container: xampp3
```

**Test from browser:** Open `https://site1.local/which.php` in separate incognito windows.

**Result:** ✅ Pass

---

### TC-08: Nginx Reachability to Backends

**Criteria:** Nginx can connect to all XAMPP backends internally.

```bash
docker exec nginx wget -qO- http://xampp1:8081
docker exec nginx wget -qO- http://xampp2:8081
docker exec nginx wget -qO- http://xampp3:8081
```

**Expected:** HTML content returned from each.

**Result:** ✅ Pass

---

### TC-09: MySQL Connectivity

**Criteria:** Each site can connect to its dedicated database.

```bash
# Test from browser
https://site1.local/db-test.php   # → "Connected to: site1_db"
```

**db-test.php:**
```php
<?php
require 'config.php';
try {
    $stmt = $pdo->query("SELECT DATABASE()");
    echo "Connected to: " . $stmt->fetchColumn();
} catch (PDOException $e) {
    echo "Error: " . $e->getMessage();
}
```

**Result:** ✅ Pass

---

### TC-10: Data Persistence

**Criteria:** MySQL data survives container restart.

```bash
docker-compose restart
docker exec -it xampp1 /opt/lampp/bin/mysql -u root -e "SHOW DATABASES;"
```

**Expected:** site1_db, site2_db, site3_db still present.

**Result:** ✅ Pass — data persisted via `xampp_data` named volume.

---

### Test Summary

| Test | Description | Result |
|------|-------------|--------|
| TC-01 | Container health | ✅ Pass |
| TC-02 | Virtual host routing | ✅ Pass |
| TC-03 | HTTP → HTTPS redirect | ✅ Pass |
| TC-04 | SSL certificate trust | ✅ Pass |
| TC-05 | Apache config validation | ✅ Pass |
| TC-06 | Virtual host registration | ✅ Pass |
| TC-07 | Load balancer distribution | ✅ Pass |
| TC-08 | Nginx backend reachability | ✅ Pass |
| TC-09 | MySQL connectivity | ✅ Pass |
| TC-10 | Data persistence | ✅ Pass |

---

## Useful Commands

```bash
# Start the stack
docker-compose up -d

# Stop the stack
docker-compose down

# Stop and remove orphan containers
docker-compose down --remove-orphans

# Restart a single service
docker-compose restart xampp1

# View logs
docker logs xampp1
docker logs nginx

# Nginx error log
docker exec nginx cat /var/log/nginx/error.log

# Reload Nginx config without restart
docker exec nginx nginx -s reload

# Validate Nginx config
docker exec nginx nginx -t

# Validate Apache config
docker exec xampp1 /opt/lampp/bin/httpd -t

# Check registered virtual hosts
docker exec xampp1 /opt/lampp/bin/httpd -S

# Check listening ports inside container
docker exec xampp1 netstat -tlnp

# Inspect Docker network
docker network inspect xampp_web

# Connect to MySQL
docker exec -it xampp1 /opt/lampp/bin/mysql -u root

# Test load balancing
for i in 1 2 3 4 5 6; do curl -sk https://site1.local/which.php; echo; done
```

---

## Troubleshooting

### 502 Bad Gateway

Nginx cannot reach XAMPP backends.

```bash
docker exec nginx cat /var/log/nginx/error.log
docker exec xampp1 /opt/lampp/bin/httpd -t    # check for config errors
docker ps                                      # confirm containers are Up
```

### All sites show the same content

Virtual hosts not loaded. Check:
```bash
# Confirm Include is uncommented in httpd.conf
docker exec xampp1 grep -n "vhosts" /opt/lampp/etc/httpd.conf
# Should show: Include etc/extra/httpd-vhosts.conf  (no leading #)
```

### XAMPP container keeps restarting

Usually a config syntax error or MySQL volume conflict.
```bash
docker logs xampp1
docker exec xampp1 /opt/lampp/bin/httpd -t
```

### SSL certificate warning in browser

mkcert CA not trusted. On Windows run as Administrator:
```powershell
mkcert -install
```
Then restart the browser completely.

### Cannot connect to MySQL from Windows

Verify port 3306 is exposed:
```bash
docker ps | grep xampp1
# Should show: 0.0.0.0:3306->3306/tcp
```

---

## Images Used

| Image | Version | Purpose |
|-------|---------|---------|
| tomsik68/xampp | latest | Apache + PHP + MySQL + phpMyAdmin |
| nginx | alpine | Reverse proxy + load balancer |

---

*Documentation generated from a hands-on build session on Ubuntu 22.04 (LAB_server) with Windows 11 as the testing client.*
