# Nginx Proxy Manager Configuration for Nick's Mining Pool

## Overview

This configuration sets up:
- **5 Stratum TCP Streams** (for mining connections)
- **1 HTTP Proxy Host** (for web interface)
- **External IP**: 108.249.26.52 (public, via Cloudflare)
- **NPM Server**: 172.16.0.21 (internal LAN)
- **Mining Pool Backend**: 172.16.111.20 (internal LAN)

## Network Architecture

```
Internet (Miners/Users)
        ↓
108.249.26.52 (External IP via Cloudflare)
        ↓
Router/Firewall (Port Forwarding)
        ↓
172.16.0.21 (NPM Server - Internal LAN)
        ↓
172.16.111.20 (Mining Pool Backend - Internal LAN)
```

## Domain & Port Mapping

### Stratum Streams (TCP)
| Coin | Domain | Port | NPM Internal IP | Backend Internal IP |
|------|--------|------|-----------------|---------------------|
| Bitcoin | btc.nicksminingpool.com | 6004 | 172.16.0.21:6004 | 172.16.111.20:6004 |
| Bitcoin Cash | bch.nicksminingpool.com | 6002 | 172.16.0.21:6002 | 172.16.111.20:6002 |
| Bitcoin SV | bsv.nicksminingpool.com | 6005 | 172.16.0.21:6005 | 172.16.111.20:6005 |
| Dogecoin | doge.nicksminingpool.com | 6003 | 172.16.0.21:6003 | 172.16.111.20:6003 |
| eCash | ecash.nicksminingpool.com | 6007 | 172.16.0.21:6007 | 172.16.111.20:6007 |

### Web Interface (HTTP/HTTPS)
| Domain | Port | NPM Internal IP | Backend Internal IP |
|--------|------|-----------------|---------------------|
| nicksminingpool.com | 80/443 | 172.16.0.21 | 172.16.111.20:8559 |

---

## Step 1: DNS Configuration (Cloudflare)

### Important: Cloudflare Proxy Settings

⚠️ **CRITICAL**: Configure Cloudflare proxy settings correctly for mining pool operation:

**Web Interface (HTTP/HTTPS):**
- ✅ **Orange Cloud (Proxied)** - nicksminingpool.com, www.nicksminingpool.com
- Provides DDoS protection, SSL, caching

**Stratum Subdomains (Mining Ports):**
- ✅ **Gray Cloud (DNS-only)** - btc, bch, bsv, doge, ecash.nicksminingpool.com
- Mining connections MUST bypass Cloudflare proxy
- Direct connection to your server required for stratum protocol

### DNS Records Configuration

Configure your Cloudflare DNS records:

```
A     nicksminingpool.com          →  108.249.26.52  [🟠 Proxied]
A     www.nicksminingpool.com      →  108.249.26.52  [🟠 Proxied]
A     btc.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only]
A     bch.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only]
A     bsv.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only]
A     doge.nicksminingpool.com     →  108.249.26.52  [⚫ DNS-only]
A     ecash.nicksminingpool.com    →  108.249.26.52  [⚫ DNS-only]
```

**Alternative (recommended):** Use wildcard DNS with manual overrides:
```
A     *.nicksminingpool.com        →  108.249.26.52  [⚫ DNS-only]
A     nicksminingpool.com          →  108.249.26.52  [🟠 Proxied]
A     www.nicksminingpool.com      →  108.249.26.52  [🟠 Proxied]
```

### Router/Firewall Port Forwarding

Configure port forwarding on your router to NPM server (172.16.0.21):

```
External IP: 108.249.26.52
↓ Forward these ports to NPM (172.16.0.21)

Port 80 (HTTP)       → 172.16.0.21:80
Port 443 (HTTPS)     → 172.16.0.21:443
Port 6002 (BCH)      → 172.16.0.21:6002
Port 6003 (DOGE)     → 172.16.0.21:6003
Port 6004 (BTC)      → 172.16.0.21:6004
Port 6005 (BSV)      → 172.16.0.21:6005
Port 6007 (XEC)      → 172.16.0.21:6007
```

---

## Step 2: Docker Compose Configuration

### Create/Edit docker-compose.yml

```bash
cd ~/nginx-proxy-manager
nano docker-compose.yml
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      # NPM Admin Interface
      - '81:81'

      # HTTP/HTTPS for web interface
      - '80:80'
      - '443:443'

      # Stratum Mining Ports
      - '6002:6002'  # Bitcoin Cash (BCH)
      - '6003:6003'  # Dogecoin (DOGE)
      - '6004:6004'  # Bitcoin (BTC)
      - '6005:6005'  # Bitcoin SV (BSV)
      - '6007:6007'  # eCash (XEC)

    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
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

### Start/Restart NPM

```bash
# If first time installing
docker-compose up -d

# If updating existing installation
docker-compose down
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f app
```

---

## Step 3: Configure Firewall

Open required ports on your NPM server:

```bash
# Allow NPM admin interface (restrict to your IP later)
sudo ufw allow 81/tcp

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow Stratum ports
sudo ufw allow 6002/tcp  # BCH
sudo ufw allow 6003/tcp  # DOGE
sudo ufw allow 6004/tcp  # BTC
sudo ufw allow 6005/tcp  # BSV
sudo ufw allow 6007/tcp  # XEC

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status numbered
```

---

## Step 4: Access NPM Web Interface

1. Open browser to: `http://YOUR_NPM_SERVER_IP:81`
2. Default credentials:
   - **Email**: `admin@example.com`
   - **Password**: `changeme`
3. **IMPORTANT:** Change these credentials immediately!

---

## Step 5: Configure Streams (Mining Ports)

### Method A: Using NPM GUI (Recommended if version 2.10.0+)

#### Bitcoin (BTC) Stream

1. In NPM web interface, click **"Streams"** tab
2. Click **"Add Stream"**
3. Configure:
   - **Incoming Port**: `6004`
   - **Forwarding Host**: `172.16.111.20`
   - **Forwarding Port**: `6004`
   - **TCP Forwarding**: ✅ Enabled
   - **UDP Forwarding**: ❌ Disabled
4. Click **"Save"**

#### Bitcoin Cash (BCH) Stream

1. Click **"Add Stream"**
2. Configure:
   - **Incoming Port**: `6002`
   - **Forwarding Host**: `172.16.111.20`
   - **Forwarding Port**: `6002`
   - **TCP Forwarding**: ✅ Enabled
   - **UDP Forwarding**: ❌ Disabled
3. Click **"Save"**

#### Bitcoin SV (BSV) Stream

1. Click **"Add Stream"**
2. Configure:
   - **Incoming Port**: `6005`
   - **Forwarding Host**: `172.16.111.20`
   - **Forwarding Port**: `6005`
   - **TCP Forwarding**: ✅ Enabled
   - **UDP Forwarding**: ❌ Disabled
3. Click **"Save"**

#### Dogecoin (DOGE) Stream

1. Click **"Add Stream"**
2. Configure:
   - **Incoming Port**: `6003`
   - **Forwarding Host**: `172.16.111.20`
   - **Forwarding Port**: `6003`
   - **TCP Forwarding**: ✅ Enabled
   - **UDP Forwarding**: ❌ Disabled
3. Click **"Save"**

#### eCash (XEC) Stream

1. Click **"Add Stream"**
2. Configure:
   - **Incoming Port**: `6007`
   - **Forwarding Host**: `172.16.111.20`
   - **Forwarding Port**: `6007`
   - **TCP Forwarding**: ✅ Enabled
   - **UDP Forwarding**: ❌ Disabled
3. Click **"Save"**

---

### Method B: Manual Configuration (If GUI Streams Not Available)

If your NPM version doesn't have the Streams GUI, configure manually:

#### Create Custom Stream Configuration

```bash
# Access NPM container
docker exec -it nginx-proxy-manager bash

# Create custom directory
mkdir -p /data/nginx/custom

# Create stratum configuration file
nano /data/nginx/custom/stratum-streams.conf
```

**stratum-streams.conf content:**
```nginx
# Bitcoin Cash (BCH) - Port 6002
stream {
    upstream bch_backend {
        server 108.249.26.52:6002 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6002;
        proxy_pass bch_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;
    }
}

# Dogecoin (DOGE) - Port 6003
stream {
    upstream doge_backend {
        server 108.249.26.52:6003 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6003;
        proxy_pass doge_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;
    }
}

# Bitcoin (BTC) - Port 6004
stream {
    upstream btc_backend {
        server 108.249.26.52:6004 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6004;
        proxy_pass btc_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;
    }
}

# Bitcoin SV (BSV) - Port 6005
stream {
    upstream bsv_backend {
        server 108.249.26.52:6005 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6005;
        proxy_pass bsv_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;
    }
}

# eCash (XEC) - Port 6007
stream {
    upstream ecash_backend {
        server 108.249.26.52:6007 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6007;
        proxy_pass ecash_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;
    }
}
```

#### Include Configuration in Main Nginx Config

```bash
# Edit main nginx.conf
nano /etc/nginx/nginx.conf
```

**Add at the end (before last closing brace):**
```nginx
# Include custom stream configurations
include /data/nginx/custom/*.conf;
```

#### Test and Reload

```bash
# Test configuration
nginx -t

# If test passes, reload nginx
nginx -s reload

# Exit container
exit
```

---

## Step 6: Configure Proxy Host (Web Interface)

### Add Proxy Host for Web UI

1. In NPM web interface, click **"Hosts"** → **"Proxy Hosts"**
2. Click **"Add Proxy Host"**
3. **Details tab**:
   - **Domain Names**: `nicksminingpool.com`, `www.nicksminingpool.com`
   - **Scheme**: `http`
   - **Forward Hostname/IP**: `172.16.111.20` (mining pool backend internal IP)
   - **Forward Port**: `8559`
   - **Cache Assets**: ✅ Enabled (optional)
   - **Block Common Exploits**: ✅ Enabled
   - **Websockets Support**: ✅ Enabled (if your web UI uses websockets)

4. **SSL tab**:
   - **SSL Certificate**: Select "Request a new SSL Certificate"
   - **Force SSL**: ✅ Enabled
   - **HTTP/2 Support**: ✅ Enabled
   - **HSTS Enabled**: ✅ Enabled (optional, recommended)
   - **Email Address**: your-email@example.com
   - **I Agree to Let's Encrypt Terms**: ✅ Checked

5. Click **"Save"**

NPM will automatically:
- Request SSL certificate from Let's Encrypt
- Configure HTTPS redirect
- Set up auto-renewal

---

## Step 7: Verify Configuration

### Test Stratum Ports

```bash
# Test Bitcoin (BTC) - Port 6004
telnet btc.nicksminingpool.com 6004

# Test Bitcoin Cash (BCH) - Port 6002
telnet bch.nicksminingpool.com 6002

# Test Bitcoin SV (BSV) - Port 6005
telnet bsv.nicksminingpool.com 6005

# Test Dogecoin (DOGE) - Port 6003
telnet doge.nicksminingpool.com 6003

# Test eCash (XEC) - Port 6007
telnet ecash.nicksminingpool.com 6007
```

**Expected:** Connection established to each port

**If connection works, test stratum protocol:**
```bash
# Connect and send stratum command
echo '{"id": 1, "method": "mining.subscribe", "params": []}' | nc btc.nicksminingpool.com 6004
```

**Expected:** JSON response from pool

### Test Web Interface

```bash
# Test HTTP (should redirect to HTTPS)
curl -I http://nicksminingpool.com

# Test HTTPS
curl -I https://nicksminingpool.com
```

**Expected:**
- HTTP returns 301 redirect to HTTPS
- HTTPS returns 200 OK

**Browser test:**
- Visit: `https://nicksminingpool.com`
- Should show your mining pool web interface
- SSL certificate should be valid (green padlock)

### Check Listening Ports

```bash
# On NPM server
sudo netstat -tlnp | grep -E '(6002|6003|6004|6005|6007|80|443)'
```

**Expected output:**
```
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1234/docker-proxy
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      1234/docker-proxy
tcp        0      0 0.0.0.0:6002            0.0.0.0:*               LISTEN      1234/docker-proxy
tcp        0      0 0.0.0.0:6003            0.0.0.0:*               LISTEN      1234/docker-proxy
tcp        0      0 0.0.0.0:6004            0.0.0.0:*               LISTEN      1234/docker-proxy
tcp        0      0 0.0.0.0:6005            0.0.0.0:*               LISTEN      1234/docker-proxy
tcp        0      0 0.0.0.0:6007            0.0.0.0:*               LISTEN      1234/docker-proxy
```

---

## Step 8: Test Mining Connections

### Connection Strings for Miners

**Bitcoin (BTC):**
```bash
cgminer -o stratum+tcp://btc.nicksminingpool.com:6004 -u WALLET_ADDRESS.worker1 -p x
```

**Bitcoin Cash (BCH):**
```bash
cgminer -o stratum+tcp://bch.nicksminingpool.com:6002 -u WALLET_ADDRESS.worker1 -p x
```

**Bitcoin SV (BSV):**
```bash
cgminer -o stratum+tcp://bsv.nicksminingpool.com:6005 -u WALLET_ADDRESS.worker1 -p x
```

**Dogecoin (DOGE):**
```bash
cgminer -o stratum+tcp://doge.nicksminingpool.com:6003 -u WALLET_ADDRESS.worker1 -p x
```

**eCash (XEC):**
```bash
cgminer -o stratum+tcp://ecash.nicksminingpool.com:6007 -u WALLET_ADDRESS.worker1 -p x
```

---

## Complete Configuration Summary

### NPM Configuration Overview

```
                    Internet (Miners/Users)
                            │
                            ↓
               ┌────────────────────────────┐
               │  108.249.26.52 (External)  │
               │  Cloudflare DNS            │
               └────────────────────────────┘
                            │
                            ↓
               ┌────────────────────────────┐
               │  Router/Firewall           │
               │  Port Forwarding           │
               └────────────────────────────┘
                            │
                            ↓
┌───────────────────────────────────────────────────────┐
│  Nginx Proxy Manager (172.16.0.21)                    │
│                                                       │
│  Streams (TCP Forwarding):                            │
│  ├─ Port 6002 → 172.16.111.20:6002 (BCH)             │
│  ├─ Port 6003 → 172.16.111.20:6003 (DOGE)            │
│  ├─ Port 6004 → 172.16.111.20:6004 (BTC)             │
│  ├─ Port 6005 → 172.16.111.20:6005 (BSV)             │
│  └─ Port 6007 → 172.16.111.20:6007 (XEC)             │
│                                                       │
│  Proxy Host (HTTP/HTTPS):                             │
│  └─ Port 80/443 → 172.16.111.20:8559                 │
│     (nicksminingpool.com)                             │
└───────────────────────────────────────────────────────┘
                            │
                            ↓
               ┌────────────────────────────┐
               │  Mining Pool Backend       │
               │  172.16.111.20 (Internal)  │
               │                            │
               │  Stratum Pools:            │
               │  ├─ BTC  :6004             │
               │  ├─ BCH  :6002             │
               │  ├─ BSV  :6005             │
               │  ├─ DOGE :6003             │
               │  └─ XEC  :6007             │
               │                            │
               │  Web UI  :8559             │
               └────────────────────────────┘
```

---

## Troubleshooting

### Issue: Cannot Access NPM Admin (Port 81)

```bash
# Check if NPM is running
docker ps | grep nginx-proxy-manager

# Check logs
docker logs nginx-proxy-manager --tail 50

# Verify port 81 is open
sudo ufw allow 81/tcp

# Restart NPM
cd ~/nginx-proxy-manager
docker-compose restart
```

### Issue: Stratum Port Not Responding

```bash
# Check if port is open
sudo ufw status | grep 6004

# Test connection to backend directly from NPM server
nc -zv 172.16.111.20 6004

# Check NPM logs
docker logs nginx-proxy-manager | grep 6004

# Verify stream configuration
docker exec -it nginx-proxy-manager cat /data/nginx/custom/stratum-streams.conf
```

### Issue: Web Interface Shows "Bad Gateway"

**Possible causes:**
1. Backend server (172.16.111.20:8559) is not running
2. Firewall blocking port 8559 on backend
3. NPM cannot reach backend IP (network routing issue)

**Solutions:**
```bash
# Test backend directly from NPM server
curl http://172.16.111.20:8559

# Check NPM proxy host configuration
# In NPM GUI: Hosts → Proxy Hosts → Edit nicksminingpool.com

# Check NPM logs
docker logs nginx-proxy-manager | grep nicksminingpool
```

### Issue: SSL Certificate Failed

```bash
# Ensure port 80 is accessible (required for Let's Encrypt)
sudo ufw allow 80/tcp

# Verify DNS is pointing to NPM server
nslookup nicksminingpool.com

# Check DNS propagation
dig nicksminingpool.com +short

# Manually request certificate
docker exec -it nginx-proxy-manager bash
certbot certonly --standalone -d nicksminingpool.com -d www.nicksminingpool.com
```

### Issue: Miners Cannot Connect

```bash
# Test each port with telnet
telnet btc.nicksminingpool.com 6004
telnet bch.nicksminingpool.com 6002
telnet bsv.nicksminingpool.com 6005
telnet doge.nicksminingpool.com 6003
telnet ecash.nicksminingpool.com 6007

# If telnet works but miners don't connect:
# - Check miner configuration (wallet address, worker name)
# - Check backend pool logs on 108.249.26.52
# - Verify stratum protocol is working (send test command)

# Test stratum protocol
echo '{"id": 1, "method": "mining.subscribe", "params": []}' | nc btc.nicksminingpool.com 6004
```

---

## Security Recommendations

### Restrict NPM Admin Access

```bash
# Only allow admin access from your IP
sudo ufw delete allow 81/tcp
sudo ufw allow from YOUR_ADMIN_IP to any port 81 proto tcp
```

### Enable Fail2Ban

```bash
# Install fail2ban
sudo apt install -y fail2ban

# Create NPM jail
sudo nano /etc/fail2ban/jail.local
```

**Add:**
```ini
[nginx-proxy-manager]
enabled = true
port = 80,443
filter = nginx-proxy-manager
logpath = /home/yourusername/nginx-proxy-manager/data/logs/*.log
maxretry = 5
bantime = 3600
```

### Regular Updates

```bash
# Update NPM
cd ~/nginx-proxy-manager
docker-compose pull
docker-compose up -d

# Update system
sudo apt update && sudo apt upgrade -y
```

### Backup Configuration

```bash
# Backup NPM configuration
cd ~/nginx-proxy-manager
tar -czf npm-backup-$(date +%Y%m%d).tar.gz data/ letsencrypt/

# Store backup safely (offsite)
```

---

## Quick Reference

### Miner Connection Strings

```
Bitcoin (BTC):
  stratum+tcp://btc.nicksminingpool.com:6004

Bitcoin Cash (BCH):
  stratum+tcp://bch.nicksminingpool.com:6002

Bitcoin SV (BSV):
  stratum+tcp://bsv.nicksminingpool.com:6005

Dogecoin (DOGE):
  stratum+tcp://doge.nicksminingpool.com:6003

eCash (XEC):
  stratum+tcp://ecash.nicksminingpool.com:6007
```

### Web Interface

```
https://nicksminingpool.com
```

### Management

```bash
# View NPM logs
docker logs -f nginx-proxy-manager

# Restart NPM
cd ~/nginx-proxy-manager && docker-compose restart

# Stop NPM
docker-compose down

# Start NPM
docker-compose up -d
```

---

## Monitoring

### Check Active Connections

```bash
# Monitor connections on stratum ports
watch 'netstat -an | grep -E "(6002|6003|6004|6005|6007)" | grep ESTABLISHED | wc -l'

# View active connections per port
netstat -an | grep ESTABLISHED | grep -E "(6002|6003|6004|6005|6007)"
```

### Monitor NPM Resource Usage

```bash
# Container stats
docker stats nginx-proxy-manager

# Detailed resource usage
docker exec nginx-proxy-manager top
```

---

**Configuration Complete!** ✅

Your Nick's Mining Pool is now accessible at:
- **Web**: https://nicksminingpool.com
- **BTC Mining**: btc.nicksminingpool.com:6004
- **BCH Mining**: bch.nicksminingpool.com:6002
- **BSV Mining**: bsv.nicksminingpool.com:6005
- **DOGE Mining**: doge.nicksminingpool.com:6003
- **XEC Mining**: ecash.nicksminingpool.com:6007

---

**Last Updated:** January 21, 2026
**Support:** Run commands in troubleshooting section or check NPM logs
