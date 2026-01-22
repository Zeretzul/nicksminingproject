# Complete Guide: Configuring Stratum Mining Pool with Nginx Proxy Manager

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Part 1: Installing the Stratum Mining Pool](#part-1-installing-the-stratum-mining-pool)
5. [Part 2: Installing Nginx Proxy Manager](#part-2-installing-nginx-proxy-manager)
6. [Part 3: Configuring NPM for Stratum Protocol](#part-3-configuring-npm-for-stratum-protocol)
7. [Part 4: SSL/TLS Configuration](#part-4-ssltls-configuration)
8. [Part 5: Advanced Configuration](#part-5-advanced-configuration)
9. [Part 6: Testing and Verification](#part-6-testing-and-verification)
10. [Part 7: Troubleshooting](#part-7-troubleshooting)
11. [Part 8: Security Best Practices](#part-8-security-best-practices)
12. [Appendix](#appendix)

---

## Overview

### What is a Stratum Mining Pool?

Stratum is a protocol used by cryptocurrency mining pools to coordinate work between miners and the pool server. It's more efficient than the older getwork protocol and is the standard for modern mining operations.

### What is Nginx Proxy Manager?

Nginx Proxy Manager (NPM) is a Docker-based tool that provides:
- Reverse proxy functionality
- SSL/TLS certificate management (via Let's Encrypt)
- Easy web-based configuration interface
- Load balancing capabilities

### Why Use NPM with Stratum?

- **SSL/TLS Support**: Encrypt miner connections for security
- **Load Balancing**: Distribute miners across multiple pool servers
- **Centralized Management**: Single interface for all proxy configurations
- **DDoS Protection**: Rate limiting and connection management
- **Multiple Coin Support**: Route different coins to different backends

---

## Prerequisites

### Hardware Requirements

**Minimum:**
- 2 CPU cores
- 4 GB RAM
- 50 GB storage
- 100 Mbps network connection

**Recommended for Production:**
- 4+ CPU cores
- 8+ GB RAM
- 100+ GB SSD storage
- 1 Gbps network connection

### Software Requirements

- Ubuntu 20.04/22.04 LTS or Debian 11/12 (recommended)
- Docker and Docker Compose
- Root or sudo access
- Domain name (for SSL certificates)
- Basic Linux command line knowledge

### Network Requirements

- Static IP address (or DDNS)
- Firewall access to open required ports
- Domain DNS configured to point to your server

### Ports Required

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| Nginx Proxy Manager Web UI | 81 | HTTP | Management interface |
| NPM HTTP | 80 | HTTP | Let's Encrypt validation |
| NPM HTTPS | 443 | HTTPS | Secure web traffic |
| Stratum (Bitcoin) | 3333 | TCP | Mining connections |
| Stratum SSL (Bitcoin) | 3334 | TCP | Secure mining connections |
| Stratum (Ethereum) | 8008 | TCP | Mining connections |
| Stratum SSL (Ethereum) | 8009 | TCP | Secure mining connections |
| Mining Pool Backend | 8080-8081 | HTTP | Pool API/Web interface |

*Note: Ports vary by cryptocurrency. Adjust as needed.*

---

## Architecture

### Network Diagram

```
Internet
    |
    v
[Firewall/Router]
    |
    v
[Nginx Proxy Manager - Port 3333, 3334]
    |
    +---> [Stratum Pool Server 1] - Port 3333 (Backend)
    |
    +---> [Stratum Pool Server 2] - Port 3333 (Backend)
    |
    v
[Redis/Database]
```

### Component Overview

1. **Nginx Proxy Manager**: Frontend proxy handling incoming connections
2. **Stratum Pool Server**: Mining pool software (NOMP, Prohashing, etc.)
3. **Coin Daemon**: Full node for cryptocurrency blockchain
4. **Redis**: Session storage and caching
5. **Database**: MySQL/PostgreSQL for pool data

---

## Part 1: Installing the Stratum Mining Pool

### Option A: Using NOMP (Node Open Mining Portal)

NOMP is a popular, open-source mining pool solution supporting multiple cryptocurrencies.

#### Step 1: Install Node.js and Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 16.x
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
sudo apt install -y nodejs

# Install build tools
sudo apt install -y build-essential libssl-dev libboost-all-dev

# Verify installation
node --version  # Should show v16.x.x
npm --version
```

#### Step 2: Install Redis

```bash
# Install Redis
sudo apt install -y redis-server

# Configure Redis for production
sudo nano /etc/redis/redis.conf
```

**Key Redis configurations:**
```
# Bind to localhost only (secure)
bind 127.0.0.1

# Set max memory
maxmemory 256mb
maxmemory-policy allkeys-lru

# Enable persistence
save 900 1
save 300 10
save 60 10000
```

```bash
# Restart Redis
sudo systemctl restart redis-server
sudo systemctl enable redis-server

# Verify Redis is running
redis-cli ping  # Should return "PONG"
```

#### Step 3: Install Coin Daemon (Example: Bitcoin)

```bash
# Download Bitcoin Core (example for Bitcoin)
cd ~
wget https://bitcoin.org/bin/bitcoin-core-25.0/bitcoin-25.0-x86_64-linux-gnu.tar.gz

# Extract
tar -xvf bitcoin-25.0-x86_64-linux-gnu.tar.gz

# Install
sudo install -m 0755 -o root -g root -t /usr/local/bin bitcoin-25.0/bin/*

# Create Bitcoin data directory
mkdir -p ~/.bitcoin

# Create bitcoin.conf
nano ~/.bitcoin/bitcoin.conf
```

**bitcoin.conf configuration:**
```
# RPC Configuration
rpcuser=yourrpcuser
rpcpassword=yourverystrongrpcpassword
rpcallowip=127.0.0.1

# Stratum Configuration
server=1
daemon=1
txindex=1

# Network
listen=1
port=8333

# RPC Port
rpcport=8332

# Logging
debug=0
```

```bash
# Start Bitcoin daemon
bitcoind -daemon

# Wait for initial sync (this can take days)
bitcoin-cli getblockchaininfo
```

#### Step 4: Install NOMP

```bash
# Clone NOMP repository
cd ~
git clone https://github.com/zone117x/node-open-mining-portal.git NOMP
cd NOMP

# Install dependencies
npm update
npm install
```

#### Step 5: Configure NOMP

```bash
# Copy example configs
cp config_example.json config.json
cp -r coins_example coins
cp -r pool_configs_example pool_configs
```

**Edit config.json:**
```bash
nano config.json
```

```json
{
    "clustering": {
        "enabled": false,
        "forks": "auto"
    },
    "defaultPoolConfigs": {
        "blockRefreshInterval": 1000,
        "jobRebroadcastTimeout": 55,
        "connectionTimeout": 600,
        "emitInvalidBlockHashes": false,
        "validateWorkerUsername": true,
        "tcpProxyProtocol": false,
        "banning": {
            "enabled": true,
            "time": 600,
            "invalidPercent": 50,
            "checkThreshold": 500,
            "purgeInterval": 300
        },
        "redis": {
            "host": "127.0.0.1",
            "port": 6379
        }
    },
    "website": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 8080,
        "stratumHost": "pool.yourdomain.com",
        "stats": {
            "updateInterval": 60,
            "historicalRetention": 43200,
            "hashrateWindow": 300
        },
        "adminCenter": {
            "enabled": true,
            "password": "yourAdminPassword"
        }
    },
    "redis": {
        "host": "127.0.0.1",
        "port": 6379
    },
    "switching": {
        "switch1": {
            "enabled": false
        }
    },
    "profitSwitch": {
        "enabled": false
    }
}
```

**Configure Bitcoin pool (pool_configs/bitcoin.json):**
```bash
nano pool_configs/bitcoin.json
```

```json
{
    "enabled": true,
    "coin": "bitcoin.json",
    "address": "YOUR_BITCOIN_ADDRESS_FOR_POOL_PAYMENTS",
    "rewardRecipients": {
        "YOUR_BITCOIN_ADDRESS": 1.0
    },
    "paymentProcessing": {
        "enabled": true,
        "paymentInterval": 600,
        "minimumPayment": 0.001,
        "maxBlocksPerPayment": 10,
        "daemon": {
            "host": "127.0.0.1",
            "port": 8332,
            "user": "yourrpcuser",
            "password": "yourverystrongrpcpassword"
        }
    },
    "ports": {
        "3333": {
            "diff": 32,
            "varDiff": {
                "minDiff": 8,
                "maxDiff": 512,
                "targetTime": 15,
                "retargetTime": 90,
                "variancePercent": 30
            }
        },
        "3334": {
            "diff": 256,
            "varDiff": {
                "minDiff": 128,
                "maxDiff": 2048,
                "targetTime": 15,
                "retargetTime": 90,
                "variancePercent": 30
            }
        }
    },
    "daemons": [
        {
            "host": "127.0.0.1",
            "port": 8332,
            "user": "yourrpcuser",
            "password": "yourverystrongrpcpassword"
        }
    ],
    "p2p": {
        "enabled": false
    }
}
```

**Configure Bitcoin coin file (coins/bitcoin.json):**
```bash
nano coins/bitcoin.json
```

```json
{
    "name": "Bitcoin",
    "symbol": "BTC",
    "algorithm": "sha256",
    "peerMagic": "f9beb4d9",
    "peerMagicTestnet": "0b110907"
}
```

#### Step 6: Start NOMP

```bash
# Start NOMP
cd ~/NOMP
node init.js
```

**Expected output:**
```
[POSIX]  Not SysV OS
[MASTER] Starting NOMP
[MASTER] Spawned 1 pool(s) on 1 thread(s)
[POOL]   Starting coin: bitcoin
[POOL]   Started for bitcoin on port 3333
```

#### Step 7: Create Systemd Service for NOMP

```bash
sudo nano /etc/systemd/system/nomp.service
```

```ini
[Unit]
Description=Node Open Mining Portal
After=network.target redis-server.service

[Service]
Type=simple
User=yourusername
WorkingDirectory=/home/yourusername/NOMP
ExecStart=/usr/bin/node init.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable nomp
sudo systemctl start nomp

# Check status
sudo systemctl status nomp
```

### Option B: Using Prohashing Pool Software

*Prohashing is a commercial solution. For open-source alternatives, stick with NOMP or similar.*

### Option C: Other Mining Pool Software

- **MPOS**: MySQL-based pool (older, PHP-based)
- **yiimp**: Multi-algorithm pool
- **open-ethereum-pool**: Ethereum-specific
- **kawpow-pool**: For Ravencoin and similar

---

## Part 2: Installing Nginx Proxy Manager

### Step 1: Install Docker and Docker Compose

```bash
# Update system
sudo apt update

# Install Docker dependencies
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add user to docker group (logout/login required)
sudo usermod -aG docker $USER

# Verify installation
docker --version
docker-compose --version
```

### Step 2: Create NPM Directory Structure

```bash
# Create directory
mkdir -p ~/nginx-proxy-manager
cd ~/nginx-proxy-manager

# Create data directories
mkdir -p data letsencrypt
```

### Step 3: Create Docker Compose File

```bash
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      # HTTP
      - '80:80'
      # HTTPS
      - '443:443'
      # Admin Web UI
      - '81:81'
      # Stratum ports (Bitcoin example)
      - '3333:3333'
      - '3334:3334'
      # Stratum ports (Ethereum example)
      - '8008:8008'
      - '8009:8009'
      # Add more ports as needed for different coins
    environment:
      # Database connection
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
      # Disable IPv6 if not needed
      DISABLE_IPV6: 'true'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: 'jc21/mariadb-aria:latest'
    container_name: nginx-proxy-manager-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 'npm'
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm'
    volumes:
      - ./data/mysql:/var/lib/mysql

networks:
  default:
    name: npm-network
```

### Step 4: Start Nginx Proxy Manager

```bash
# Start containers
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f app
```

### Step 5: Access NPM Web Interface

1. Open browser to: `http://YOUR_SERVER_IP:81`
2. Default login credentials:
   - **Email**: `admin@example.com`
   - **Password**: `changeme`
3. Change credentials immediately after first login

---

## Part 3: Configuring NPM for Stratum Protocol

### Understanding Stratum Proxy Configuration

NPM uses Nginx streams module for TCP/UDP proxying. Since Stratum is a TCP-based protocol (not HTTP), we need to configure TCP streams.

### Step 1: Create Custom Nginx Configuration for Stratum

```bash
# Access NPM container
docker exec -it nginx-proxy-manager bash

# Create custom stream configuration directory
mkdir -p /data/nginx/custom

# Create stratum configuration file
nano /data/nginx/custom/stratum.conf
```

**stratum.conf content:**
```nginx
# Stratum TCP Stream Configuration

# Bitcoin Stratum (Port 3333 - Non-SSL)
stream {
    upstream bitcoin_stratum_backend {
        # Load balancing method: least_conn, hash, ip_hash
        least_conn;

        # Backend stratum servers
        server 127.0.0.1:3333 max_fails=3 fail_timeout=30s;
        # Add more backend servers for load balancing:
        # server 192.168.1.101:3333 max_fails=3 fail_timeout=30s;
        # server 192.168.1.102:3333 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 3333;
        proxy_pass bitcoin_stratum_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;

        # Enable TCP keepalive
        proxy_socket_keepalive on;

        # Preserve client IP
        proxy_protocol off;
    }
}

# Bitcoin Stratum SSL (Port 3334 - SSL)
stream {
    upstream bitcoin_stratum_ssl_backend {
        least_conn;
        server 127.0.0.1:3333 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 3334 ssl;
        proxy_pass bitcoin_stratum_ssl_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;

        # SSL Configuration
        ssl_certificate /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/pool.yourdomain.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
    }
}

# Ethereum Stratum (Port 8008 - Non-SSL)
stream {
    upstream ethereum_stratum_backend {
        least_conn;
        server 127.0.0.1:8008 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 8008;
        proxy_pass ethereum_stratum_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;
    }
}

# Ethereum Stratum SSL (Port 8009 - SSL)
stream {
    upstream ethereum_stratum_ssl_backend {
        least_conn;
        server 127.0.0.1:8008 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 8009 ssl;
        proxy_pass ethereum_stratum_ssl_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;

        # SSL Configuration
        ssl_certificate /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/pool.yourdomain.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
    }
}
```

```bash
# Exit container
exit
```

### Step 2: Include Custom Configuration in NPM

```bash
# Edit main nginx configuration
docker exec -it nginx-proxy-manager bash

# Edit nginx.conf to include custom configs
nano /etc/nginx/nginx.conf
```

**Add at the end of nginx.conf (before the last closing brace):**
```nginx
# Include custom stream configurations
include /data/nginx/custom/*.conf;
```

```bash
# Test configuration
nginx -t

# Exit container
exit

# Restart NPM
docker-compose restart app
```

### Step 3: Alternative Method - Using NPM Streams (GUI Method)

NPM version 2.10.0+ supports TCP/UDP streams via GUI:

1. Log into NPM web interface (port 81)
2. Navigate to **Streams** tab
3. Click **Add Stream**
4. Configure:
   - **Incoming Port**: 3333
   - **Forwarding Host**: 127.0.0.1 (or your stratum server IP)
   - **Forwarding Port**: 3333
   - **TCP Forwarding**: Enabled
   - **UDP Forwarding**: Disabled
5. Save

**Repeat for each stratum port:**
- Port 3333 → Backend 3333 (Bitcoin non-SSL)
- Port 3334 → Backend 3333 with SSL (Bitcoin SSL)
- Port 8008 → Backend 8008 (Ethereum non-SSL)
- Port 8009 → Backend 8008 with SSL (Ethereum SSL)

---

## Part 4: SSL/TLS Configuration

### Step 1: Obtain SSL Certificate via Let's Encrypt

**Method A: Using NPM GUI (Recommended)**

1. Log into NPM (http://YOUR_IP:81)
2. Go to **SSL Certificates**
3. Click **Add SSL Certificate**
4. Select **Let's Encrypt**
5. Fill in:
   - **Domain Names**: pool.yourdomain.com
   - **Email**: your-email@example.com
   - **Use a DNS Challenge**: No (unless port 80 is blocked)
   - **Agree to Terms**: Yes
6. Click **Save**

NPM will automatically:
- Request certificate from Let's Encrypt
- Validate domain ownership
- Install certificate
- Set up auto-renewal

**Method B: Manual Certbot**

```bash
# Install certbot
sudo apt install -y certbot

# Stop NPM temporarily (port 80 needed)
cd ~/nginx-proxy-manager
docker-compose down

# Request certificate
sudo certbot certonly --standalone -d pool.yourdomain.com

# Restart NPM
docker-compose up -d
```

### Step 2: Configure Stratum Ports with SSL

**Update stratum.conf with correct SSL paths:**

```bash
docker exec -it nginx-proxy-manager bash
nano /data/nginx/custom/stratum.conf
```

**Verify SSL certificate paths match:**
```nginx
ssl_certificate /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/pool.yourdomain.com/privkey.pem;
```

```bash
# Test and reload
nginx -t
nginx -s reload
exit
```

### Step 3: Configure Certificate Auto-Renewal

NPM handles auto-renewal automatically. To verify:

```bash
# Check NPM renewal job
docker exec -it nginx-proxy-manager bash
cat /etc/cron.d/certbot-renew
```

Should show renewal check running daily.

---

## Part 5: Advanced Configuration

### Load Balancing Multiple Pool Servers

**Edit stratum.conf to add multiple backends:**

```nginx
stream {
    upstream bitcoin_stratum_backend {
        # Least connections algorithm
        least_conn;

        # Pool Server 1
        server 192.168.1.101:3333 weight=1 max_fails=3 fail_timeout=30s;

        # Pool Server 2
        server 192.168.1.102:3333 weight=1 max_fails=3 fail_timeout=30s;

        # Pool Server 3
        server 192.168.1.103:3333 weight=1 max_fails=3 fail_timeout=30s;

        # Backup server (only used if all primary servers fail)
        server 192.168.1.104:3333 backup;
    }

    server {
        listen 3333;
        proxy_pass bitcoin_stratum_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;

        # Enable health checks (Nginx Plus only)
        # health_check interval=10s fails=3 passes=2;
    }
}
```

### Connection Limiting and DDoS Protection

**Add rate limiting:**

```nginx
# Define connection limit zone (add to top of stratum.conf)
limit_conn_zone $binary_remote_addr zone=stratum_conn_limit:10m;

stream {
    upstream bitcoin_stratum_backend {
        least_conn;
        server 127.0.0.1:3333;
    }

    server {
        listen 3333;
        proxy_pass bitcoin_stratum_backend;

        # Limit connections per IP
        limit_conn stratum_conn_limit 10;

        # Timeouts
        proxy_timeout 300s;
        proxy_connect_timeout 10s;

        # Enable TCP keepalive
        proxy_socket_keepalive on;
    }
}
```

### GeoIP Blocking (Optional)

```bash
# Install GeoIP module
docker exec -it nginx-proxy-manager bash
apt update
apt install -y libnginx-mod-stream-geoip

# Download GeoIP database
mkdir -p /usr/share/GeoIP
cd /usr/share/GeoIP
wget https://github.com/maxmind/geoipupdate/releases/download/v4.11.0/geoipupdate_4.11.0_linux_amd64.tar.gz
```

**Add to stratum.conf:**
```nginx
# Load GeoIP database
geoip_country /usr/share/GeoIP/GeoIP.dat;

stream {
    # Block specific countries
    map $geoip_country_code $blocked_country {
        default 0;
        CN 1;  # China
        RU 1;  # Russia
        # Add more as needed
    }

    server {
        listen 3333;

        # Deny blocked countries
        if ($blocked_country) {
            return 403;
        }

        proxy_pass bitcoin_stratum_backend;
    }
}
```

### Logging Configuration

**Configure access and error logs:**

```nginx
stream {
    # Custom log format
    log_format stratum '$remote_addr [$time_local] '
                      '$protocol $status $bytes_sent $bytes_received '
                      '$session_time "$upstream_addr" '
                      '"$upstream_bytes_sent" "$upstream_bytes_received"';

    # Access log
    access_log /var/log/nginx/stratum_access.log stratum;
    error_log /var/log/nginx/stratum_error.log warn;

    upstream bitcoin_stratum_backend {
        server 127.0.0.1:3333;
    }

    server {
        listen 3333;
        proxy_pass bitcoin_stratum_backend;

        # Enable detailed logging
        error_log /var/log/nginx/stratum_3333_error.log debug;
    }
}
```

### Monitoring with Prometheus (Optional)

```bash
# Add nginx-prometheus-exporter
nano docker-compose.yml
```

**Add to docker-compose.yml:**
```yaml
  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx-exporter
    restart: unless-stopped
    ports:
      - '9113:9113'
    command:
      - '-nginx.scrape-uri=http://app:81/stub_status'
    depends_on:
      - app
```

---

## Part 6: Testing and Verification

### Test 1: Verify NPM is Running

```bash
# Check container status
docker ps | grep nginx-proxy-manager

# Check logs
docker logs nginx-proxy-manager --tail 50

# Check listening ports
sudo netstat -tlnp | grep -E '(80|81|443|3333|3334)'
```

**Expected output:**
```
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      12345/docker-proxy
tcp        0      0 0.0.0.0:81              0.0.0.0:*               LISTEN      12345/docker-proxy
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      12345/docker-proxy
tcp        0      0 0.0.0.0:3333            0.0.0.0:*               LISTEN      12345/docker-proxy
tcp        0      0 0.0.0.0:3334            0.0.0.0:*               LISTEN      12345/docker-proxy
```

### Test 2: Test Stratum Connection with Telnet

```bash
# Test non-SSL stratum port
telnet pool.yourdomain.com 3333
```

**Expected:** Connection established, then you can type:
```json
{"id": 1, "method": "mining.subscribe", "params": []}
```

Press Enter. You should receive a JSON response from the pool.

**Test SSL stratum port:**
```bash
# Install openssl if needed
sudo apt install -y openssl

# Test SSL connection
openssl s_client -connect pool.yourdomain.com:3334 -showcerts
```

**Expected:** SSL handshake successful, certificate details displayed.

### Test 3: Test with Mining Software

**Example: CGMiner (Bitcoin)**

```bash
# Install cgminer (or use your preferred miner)
# Download from: https://github.com/ckolivas/cgminer

# Test connection
cgminer -o stratum+tcp://pool.yourdomain.com:3333 -u YOUR_WALLET_ADDRESS -p x

# Test SSL connection
cgminer -o stratum+ssl://pool.yourdomain.com:3334 -u YOUR_WALLET_ADDRESS -p x
```

**Example: Ethminer (Ethereum)**

```bash
# Test connection
ethminer -P stratum+tcp://YOUR_WALLET_ADDRESS@pool.yourdomain.com:8008

# Test SSL connection
ethminer -P stratum+ssl://YOUR_WALLET_ADDRESS@pool.yourdomain.com:8009
```

### Test 4: Verify Load Balancing (If Configured)

```bash
# Watch Nginx access logs
docker exec -it nginx-proxy-manager tail -f /var/log/nginx/stratum_access.log

# Connect multiple miners and verify distribution across backends
```

### Test 5: SSL Certificate Validation

```bash
# Check certificate expiry
echo | openssl s_client -connect pool.yourdomain.com:3334 2>/dev/null | openssl x509 -noout -dates

# Verify certificate chain
echo | openssl s_client -connect pool.yourdomain.com:3334 -showcerts 2>/dev/null | grep -A 2 "Verify return code"
```

**Expected:** "Verify return code: 0 (ok)"

---

## Part 7: Troubleshooting

### Issue 1: Cannot Access NPM Web Interface

**Symptoms:** Cannot access http://SERVER_IP:81

**Solutions:**

```bash
# Check if container is running
docker ps | grep nginx-proxy-manager

# Check container logs
docker logs nginx-proxy-manager

# Check if port 81 is accessible
sudo ufw status
sudo ufw allow 81/tcp

# Restart container
docker-compose restart app
```

### Issue 2: Stratum Port Not Responding

**Symptoms:** `telnet pool.yourdomain.com 3333` times out

**Solutions:**

```bash
# Check if port is open in firewall
sudo ufw allow 3333/tcp
sudo ufw allow 3334/tcp

# Check if Nginx is listening
docker exec -it nginx-proxy-manager netstat -tlnp | grep 3333

# Check Nginx configuration syntax
docker exec -it nginx-proxy-manager nginx -t

# Check Nginx error logs
docker exec -it nginx-proxy-manager tail -f /var/log/nginx/error.log

# Verify backend is running
nc -zv 127.0.0.1 3333

# Check if NOMP is running
sudo systemctl status nomp
```

### Issue 3: SSL Connection Fails

**Symptoms:** `openssl s_client -connect pool.yourdomain.com:3334` fails

**Solutions:**

```bash
# Verify certificate exists
docker exec -it nginx-proxy-manager ls -la /etc/letsencrypt/live/pool.yourdomain.com/

# Check certificate permissions
docker exec -it nginx-proxy-manager cat /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem

# Verify SSL configuration in stratum.conf
docker exec -it nginx-proxy-manager cat /data/nginx/custom/stratum.conf | grep ssl

# Test SSL manually
openssl s_client -connect pool.yourdomain.com:3334 -debug

# Regenerate certificate
# Via NPM GUI: SSL Certificates → Edit → Renew
```

### Issue 4: Miners Cannot Connect

**Symptoms:** Mining software shows "Connection refused" or "Stratum connection failed"

**Solutions:**

```bash
# Test stratum protocol manually
telnet pool.yourdomain.com 3333

# Send test stratum command
echo '{"id": 1, "method": "mining.subscribe", "params": []}' | nc pool.yourdomain.com 3333

# Check NOMP logs
sudo journalctl -u nomp -f

# Verify backend stratum server is responding
curl -X POST http://127.0.0.1:8080/api/stats

# Check for IP banning (if enabled in NOMP)
redis-cli keys "*ban*"
```

### Issue 5: Load Balancing Not Working

**Symptoms:** All traffic goes to one backend server

**Solutions:**

```bash
# Verify upstream configuration
docker exec -it nginx-proxy-manager cat /data/nginx/custom/stratum.conf | grep -A 10 "upstream"

# Check backend server health
nc -zv 192.168.1.101 3333
nc -zv 192.168.1.102 3333

# Monitor load distribution
docker exec -it nginx-proxy-manager tail -f /var/log/nginx/stratum_access.log

# Verify load balancing method
# Change from "least_conn" to "ip_hash" or "hash $remote_addr consistent"
```

### Issue 6: High Memory Usage

**Symptoms:** NPM container using excessive RAM

**Solutions:**

```bash
# Check container resource usage
docker stats nginx-proxy-manager

# Limit container memory (edit docker-compose.yml)
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    mem_limit: 512m
    memswap_limit: 1g

# Restart container
docker-compose up -d

# Optimize Nginx worker processes
docker exec -it nginx-proxy-manager nano /etc/nginx/nginx.conf
# Set: worker_processes 2;
# Set: worker_connections 1024;
```

### Issue 7: Certificate Auto-Renewal Fails

**Symptoms:** SSL certificate expires

**Solutions:**

```bash
# Manually renew certificate via NPM GUI
# SSL Certificates → Select cert → Renew

# Check renewal logs
docker logs nginx-proxy-manager | grep -i certbot

# Verify port 80 is accessible (required for Let's Encrypt)
curl http://pool.yourdomain.com/.well-known/acme-challenge/test

# Test renewal manually
docker exec -it nginx-proxy-manager certbot renew --dry-run
```

### Issue 8: Connection Drops / Timeouts

**Symptoms:** Miners disconnect frequently

**Solutions:**

```bash
# Increase proxy timeout in stratum.conf
proxy_timeout 600s;  # Increase from 300s to 600s
proxy_connect_timeout 30s;  # Increase from 10s

# Enable TCP keepalive
proxy_socket_keepalive on;

# Check for network issues
ping pool.yourdomain.com
mtr pool.yourdomain.com

# Monitor connection states
watch 'netstat -an | grep 3333 | grep ESTABLISHED | wc -l'
```

---

## Part 8: Security Best Practices

### Firewall Configuration

```bash
# Install UFW (Ubuntu)
sudo apt install -y ufw

# Default deny
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (IMPORTANT: Do this first!)
sudo ufw allow 22/tcp

# Allow NPM ports
sudo ufw allow 80/tcp    # HTTP (Let's Encrypt)
sudo ufw allow 81/tcp    # NPM Web UI (restrict to admin IPs)
sudo ufw allow 443/tcp   # HTTPS

# Allow stratum ports
sudo ufw allow 3333/tcp  # Bitcoin stratum
sudo ufw allow 3334/tcp  # Bitcoin stratum SSL
sudo ufw allow 8008/tcp  # Ethereum stratum
sudo ufw allow 8009/tcp  # Ethereum stratum SSL

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

**Restrict NPM admin interface to specific IPs:**
```bash
# Allow only from specific IP
sudo ufw delete allow 81/tcp
sudo ufw allow from YOUR_ADMIN_IP to any port 81 proto tcp
```

### Fail2Ban for DDoS Protection

```bash
# Install fail2ban
sudo apt install -y fail2ban

# Create custom jail for Nginx
sudo nano /etc/fail2ban/jail.local
```

```ini
[nginx-stratum]
enabled = true
port = 3333,3334,8008,8009
filter = nginx-stratum
logpath = /var/log/nginx/stratum_access.log
maxretry = 10
findtime = 300
bantime = 3600
```

```bash
# Create filter
sudo nano /etc/fail2ban/filter.d/nginx-stratum.conf
```

```ini
[Definition]
failregex = ^<HOST> .* 5\d{2}
ignoreregex =
```

```bash
# Restart fail2ban
sudo systemctl restart fail2ban

# Check status
sudo fail2ban-client status nginx-stratum
```

### Regular Security Updates

```bash
# Create update script
nano ~/update-pool-server.sh
```

```bash
#!/bin/bash

# Update system packages
sudo apt update
sudo apt upgrade -y

# Update Docker images
cd ~/nginx-proxy-manager
docker-compose pull
docker-compose up -d

# Update NOMP
cd ~/NOMP
git pull
npm update

# Restart services
sudo systemctl restart nomp

# Clean up
sudo apt autoremove -y
docker system prune -f

echo "Update completed: $(date)" >> ~/update.log
```

```bash
# Make executable
chmod +x ~/update-pool-server.sh

# Schedule weekly updates (every Sunday at 2 AM)
crontab -e

# Add line:
0 2 * * 0 /home/yourusername/update-pool-server.sh
```

### Backup Strategy

```bash
# Create backup script
nano ~/backup-pool.sh
```

```bash
#!/bin/bash

BACKUP_DIR="/backup/pool-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup NPM configuration
docker exec nginx-proxy-manager tar czf /tmp/npm-backup.tar.gz /data /etc/letsencrypt
docker cp nginx-proxy-manager:/tmp/npm-backup.tar.gz $BACKUP_DIR/

# Backup NOMP configuration
tar czf $BACKUP_DIR/nomp-config.tar.gz ~/NOMP/config.json ~/NOMP/pool_configs/ ~/NOMP/coins/

# Backup database
docker exec nginx-proxy-manager-db mysqldump -u npm -pnpm npm > $BACKUP_DIR/npm-database.sql

# Backup Redis
redis-cli --rdb $BACKUP_DIR/redis-dump.rdb

# Clean old backups (keep 30 days)
find /backup/ -type d -mtime +30 -exec rm -rf {} \;

echo "Backup completed: $(date)" >> ~/backup.log
```

```bash
# Make executable
chmod +x ~/backup-pool.sh

# Schedule daily backups (3 AM)
crontab -e

# Add line:
0 3 * * * /home/yourusername/backup-pool.sh
```

### Monitoring and Alerts

**Install monitoring stack:**

```bash
# Create monitoring directory
mkdir -p ~/monitoring
cd ~/monitoring

# Create docker-compose.yml for Prometheus + Grafana
nano docker-compose-monitoring.yml
```

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - '9090:9090'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - '3000:3000'
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=yourpassword
    volumes:
      - grafana_data:/var/lib/grafana

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - '9100:9100'

volumes:
  prometheus_data:
  grafana_data:
```

```bash
# Create Prometheus config
nano prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
```

```bash
# Start monitoring stack
docker-compose -f docker-compose-monitoring.yml up -d

# Access Grafana: http://YOUR_IP:3000
# Login: admin / yourpassword
```

### SSL/TLS Best Practices

**Harden SSL configuration:**

```nginx
# In stratum.conf SSL sections, update:

ssl_protocols TLSv1.3;  # Only use TLS 1.3
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers off;

# Enable OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/pool.yourdomain.com/chain.pem;

# HSTS (HTTP Strict Transport Security)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Session resumption
ssl_session_cache shared:SSL:50m;
ssl_session_timeout 1d;
ssl_session_tickets off;
```

---

## Appendix

### A. Common Stratum Ports by Cryptocurrency

| Cryptocurrency | Non-SSL Port | SSL Port | Algorithm |
|---------------|--------------|----------|-----------|
| Bitcoin (BTC) | 3333 | 3334 | SHA-256 |
| Ethereum (ETH) | 8008 | 8009 | Ethash |
| Litecoin (LTC) | 3333 | 3334 | Scrypt |
| Monero (XMR) | 3333 | 3334 | RandomX |
| Ravencoin (RVN) | 3636 | 3637 | KawPow |
| Zcash (ZEC) | 3333 | 3334 | Equihash |
| Dogecoin (DOGE) | 3333 | 3334 | Scrypt |
| Bitcoin Cash (BCH) | 3333 | 3334 | SHA-256 |

### B. Useful Commands Reference

```bash
# Docker management
docker-compose ps                     # List containers
docker-compose logs -f app            # View NPM logs
docker-compose restart app            # Restart NPM
docker-compose down                   # Stop all containers
docker-compose up -d                  # Start containers

# Nginx commands (inside container)
docker exec -it nginx-proxy-manager nginx -t          # Test config
docker exec -it nginx-proxy-manager nginx -s reload   # Reload config
docker exec -it nginx-proxy-manager nginx -V          # Show version

# NOMP management
sudo systemctl status nomp            # Check NOMP status
sudo systemctl restart nomp           # Restart NOMP
sudo journalctl -u nomp -f            # View NOMP logs

# Redis management
redis-cli ping                        # Test Redis
redis-cli info stats                  # View stats
redis-cli keys "*"                    # List all keys
redis-cli flushall                    # Clear all data (DANGER!)

# Network diagnostics
netstat -tlnp | grep 3333            # Check listening ports
ss -tunap | grep 3333                # Alternative to netstat
tcpdump -i any port 3333             # Capture stratum traffic
lsof -i :3333                        # See what's using port

# SSL/Certificate management
certbot certificates                  # List certificates
certbot renew                        # Renew certificates
certbot delete --cert-name pool.yourdomain.com  # Delete cert

# Monitoring
htop                                 # Process monitor
iotop                                # Disk I/O monitor
iftop                                # Network monitor
```

### C. Stratum Protocol Basics

**Stratum Message Format:**
```json
{
  "id": 1,
  "method": "mining.subscribe",
  "params": []
}
```

**Common Stratum Methods:**
- `mining.subscribe` - Subscribe to mining pool
- `mining.authorize` - Authenticate worker
- `mining.set_difficulty` - Set mining difficulty
- `mining.notify` - New work notification
- `mining.submit` - Submit share

**Example Mining Session:**
```json
# Miner → Pool: Subscribe
{"id": 1, "method": "mining.subscribe", "params": ["cgminer/4.10.0"]}

# Pool → Miner: Subscribe response
{"id": 1, "result": [["mining.notify", "subscription_id"], "extranonce1", 4], "error": null}

# Miner → Pool: Authorize
{"id": 2, "method": "mining.authorize", "params": ["worker_name", "password"]}

# Pool → Miner: Authorize response
{"id": 2, "result": true, "error": null}

# Pool → Miner: Set difficulty
{"id": null, "method": "mining.set_difficulty", "params": [16]}

# Pool → Miner: New job
{"id": null, "method": "mining.notify", "params": ["job_id", "prevhash", "coinbase1", "coinbase2", ["merkle_branch"], "version", "nbits", "ntime", true]}

# Miner → Pool: Submit share
{"id": 4, "method": "mining.submit", "params": ["worker_name", "job_id", "extranonce2", "ntime", "nonce"]}
```

### D. Performance Tuning

**Nginx Performance Settings:**

```nginx
# In nginx.conf (inside container)
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 10000;
    use epoll;
    multi_accept on;
}

stream {
    # Connection pool
    upstream bitcoin_stratum_backend {
        least_conn;
        server 127.0.0.1:3333 max_conns=5000;
        keepalive 100;
    }
}
```

**System Optimization:**

```bash
# Increase file descriptors
sudo nano /etc/security/limits.conf

# Add:
* soft nofile 65535
* hard nofile 65535

# Increase network buffer sizes
sudo nano /etc/sysctl.conf

# Add:
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_congestion_control = bbr

# Apply changes
sudo sysctl -p

# Increase connection tracking
sudo nano /etc/sysctl.conf

# Add:
net.netfilter.nf_conntrack_max = 1048576
net.nf_conntrack_max = 1048576

# Apply
sudo sysctl -p
```

### E. Alternative Mining Pool Software

**NOMP Alternatives:**

1. **MPOS (PHP-based)**
   - GitHub: https://github.com/MPOS/php-mpos
   - Language: PHP
   - Database: MySQL
   - Best for: Traditional web hosting

2. **yiimp**
   - GitHub: https://github.com/tpruvot/yiimp
   - Multi-algorithm support
   - Best for: Multi-coin pools

3. **open-ethereum-pool**
   - GitHub: https://github.com/sammy007/open-ethereum-pool
   - Ethereum-specific
   - High performance

4. **Prohashing (Commercial)**
   - https://prohashing.com
   - Profit-switching
   - Managed service

### F. Monitoring Dashboard Examples

**Grafana Dashboard for Pool Monitoring:**

Metrics to track:
- Active miners (current connections)
- Hashrate (pool total and per-worker)
- Shares (valid, invalid, stale)
- Blocks found
- Payouts processed
- Network latency
- Server CPU/RAM/disk usage
- Nginx connection count

**Sample Prometheus Query:**
```promql
# Active connections on port 3333
sum(rate(nginx_connections_active{port="3333"}[5m]))

# Request rate
rate(nginx_http_requests_total[5m])

# Error rate
rate(nginx_http_requests_total{status=~"5.."}[5m])
```

### G. Troubleshooting Decision Tree

```
Issue: Miners cannot connect
├─ Can you telnet to port? → NO
│  ├─ Firewall blocking? → Check UFW rules
│  ├─ Nginx not listening? → Check docker-compose ports
│  └─ DNS not resolving? → Check domain configuration
│
└─ Can telnet, but no stratum response? → YES
   ├─ Backend down? → Check NOMP status
   ├─ Proxy misconfigured? → Check stratum.conf
   └─ Authentication failing? → Check worker credentials

Issue: SSL connection fails
├─ Certificate expired? → Renew via NPM or certbot
├─ Certificate not found? → Check paths in stratum.conf
├─ Wrong domain? → Regenerate for correct domain
└─ SSL protocol mismatch? → Update ssl_protocols

Issue: High latency / disconnects
├─ Network congestion? → Check bandwidth usage
├─ Timeout too short? → Increase proxy_timeout
├─ Backend overloaded? → Add more pool servers
└─ DDoS attack? → Enable rate limiting
```

### H. License and Credits

This guide is provided as-is for educational purposes.

**Software Credits:**
- Nginx Proxy Manager: https://nginxproxymanager.com
- NOMP: https://github.com/zone117x/node-open-mining-portal
- Redis: https://redis.io
- Docker: https://docker.com

**Useful Resources:**
- Stratum Protocol: https://en.bitcoin.it/wiki/Stratum_mining_protocol
- Nginx Stream Module: https://nginx.org/en/docs/stream/ngx_stream_core_module.html
- Let's Encrypt: https://letsencrypt.org

---

**Document Version:** 1.0
**Last Updated:** January 21, 2026
**Author:** Mining Pool Administrator
**Contact:** support@yourdomain.com

---

## Quick Start Summary

**For the impatient, here's the absolute minimum:**

```bash
# 1. Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh

# 2. Create NPM directory
mkdir ~/nginx-proxy-manager && cd ~/nginx-proxy-manager

# 3. Create docker-compose.yml (copy from Part 2, Step 3)

# 4. Start NPM
docker-compose up -d

# 5. Access NPM web UI
# http://YOUR_IP:81 (admin@example.com / changeme)

# 6. Install NOMP (follow Part 1)

# 7. Configure stratum proxy (Part 3)

# 8. Get SSL cert (Part 4)

# 9. Test connection
telnet pool.yourdomain.com 3333

# Done!
```

**Total setup time:** 2-4 hours (excluding blockchain sync)

---

**END OF GUIDE**
