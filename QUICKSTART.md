# Nick's Mining Pool - Quick Start Guide

## 🚀 5-Minute Setup

Follow these steps to get your mining pool proxy running.

---

## Prerequisites

- Ubuntu 20.04/22.04 or Debian 11/12
- Root or sudo access
- Docker and Docker Compose installed
- DNS records pointing to your server

---

## Step 1: Install Docker (if not already installed)

```bash
# Quick Docker installation
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add your user to docker group
sudo usermod -aG docker $USER

# Verify installation
docker --version
docker-compose --version
```

**⚠️ IMPORTANT:** Log out and log back in for group changes to take effect.

---

## Step 2: Deploy Nginx Proxy Manager

```bash
# Create project directory
mkdir -p ~/nginx-proxy-manager
cd ~/nginx-proxy-manager

# Copy the docker-compose.yml file here
# (Upload the docker-compose.yml from this repository)

# Start NPM
docker-compose up -d

# Check if running
docker-compose ps

# View logs
docker-compose logs -f
```

**Expected output:**
```
nginx-proxy-manager    ... Up    0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, ...
nginx-proxy-manager-db ... Up    3306/tcp
```

---

## Step 3: Configure Firewall

```bash
# Allow NPM admin interface
sudo ufw allow 81/tcp

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow all stratum ports
sudo ufw allow 6002/tcp  # BCH
sudo ufw allow 6003/tcp  # DOGE
sudo ufw allow 6004/tcp  # BTC
sudo ufw allow 6005/tcp  # BSV
sudo ufw allow 6007/tcp  # XEC

# Enable firewall
sudo ufw enable

# Verify
sudo ufw status
```

---

## Step 4: Access NPM Admin Panel

1. Open browser: `http://YOUR_SERVER_IP:81`

2. **Default Login:**
   - Email: `admin@example.com`
   - Password: `changeme`

3. **⚠️ IMMEDIATELY change the password!**

---

## Step 5: Configure Streams (Mining Ports)

### Option A: Using GUI (if NPM version 2.10.0+)

For each coin, add a stream:

#### Bitcoin (BTC)
1. Click **Streams** tab
2. Click **Add Stream**
3. Fill in:
   - Incoming Port: `6004`
   - Forwarding Host: `172.16.111.20`
   - Forwarding Port: `6004`
   - TCP Forwarding: ✅
   - UDP Forwarding: ❌
4. Save

#### Repeat for other coins:
- **BCH**: Incoming `6002` → `172.16.111.20:6002`
- **DOGE**: Incoming `6003` → `172.16.111.20:6003`
- **BSV**: Incoming `6005` → `172.16.111.20:6005`
- **XEC**: Incoming `6007` → `172.16.111.20:6007`

---

### Option B: Manual Configuration (if no Streams GUI)

```bash
# Copy stratum-streams.conf to container
docker cp stratum-streams.conf nginx-proxy-manager:/data/nginx/custom/

# Edit nginx.conf to include custom configs
docker exec -it nginx-proxy-manager bash

# Open nginx.conf
nano /etc/nginx/nginx.conf

# Add at the very end (before last closing brace):
# include /data/nginx/custom/*.conf;

# Save and exit (Ctrl+X, Y, Enter)

# Test configuration
nginx -t

# Reload nginx
nginx -s reload

# Exit container
exit
```

---

## Step 6: Configure Web Interface Proxy Host

1. In NPM Admin, click **Hosts** → **Proxy Hosts**
2. Click **Add Proxy Host**

### Details Tab:
- **Domain Names**: `nicksminingpool.com`, `www.nicksminingpool.com`
- **Scheme**: `http`
- **Forward Hostname/IP**: `172.16.111.20`
- **Forward Port**: `8559`
- ✅ Block Common Exploits
- ✅ Websockets Support

### SSL Tab:
- Select **"Request a new SSL Certificate"**
- ✅ Force SSL
- ✅ HTTP/2 Support
- **Email**: your-email@example.com
- ✅ Agree to Let's Encrypt Terms

3. Click **Save**

NPM will automatically request and configure SSL certificate.

---

## Step 7: Verify Everything Works

### Test Stratum Ports

```bash
# Test each stratum port
telnet btc.nicksminingpool.com 6004
telnet bch.nicksminingpool.com 6002
telnet bsv.nicksminingpool.com 6005
telnet doge.nicksminingpool.com 6003
telnet ecash.nicksminingpool.com 6007
```

**Expected:** Connection established

Press `Ctrl+]` then type `quit` to exit telnet.

### Test Web Interface

```bash
# Should redirect to HTTPS
curl -I http://nicksminingpool.com

# Should return 200 OK
curl -I https://nicksminingpool.com
```

Open in browser: `https://nicksminingpool.com`

---

## Step 8: Configure Cloudflare DNS & Port Forwarding

### Configure Router Port Forwarding FIRST

**On your router/firewall**, forward these ports to NPM server (172.16.0.21):

```
External IP: 108.249.26.52 → NPM Server: 172.16.0.21

Port 80 (HTTP)       → 172.16.0.21:80
Port 443 (HTTPS)     → 172.16.0.21:443
Port 6002 (BCH)      → 172.16.0.21:6002
Port 6003 (DOGE)     → 172.16.0.21:6003
Port 6004 (BTC)      → 172.16.0.21:6004
Port 6005 (BSV)      → 172.16.0.21:6005
Port 6007 (XEC)      → 172.16.0.21:6007
```

### Configure Cloudflare DNS

⚠️ **IMPORTANT**: Configure proxy settings correctly!

**Log into Cloudflare**, add these DNS records:

```
A     nicksminingpool.com          →  108.249.26.52  [🟠 Proxied - Click to Enable]
A     www.nicksminingpool.com      →  108.249.26.52  [🟠 Proxied - Click to Enable]
A     btc.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only - Click to Disable Proxy]
A     bch.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only - Click to Disable Proxy]
A     bsv.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only - Click to Disable Proxy]
A     doge.nicksminingpool.com     →  108.249.26.52  [⚫ DNS-only - Click to Disable Proxy]
A     ecash.nicksminingpool.com    →  108.249.26.52  [⚫ DNS-only - Click to Disable Proxy]
```

**Why different settings?**
- **Orange Cloud (Proxied)**: Web interface gets DDoS protection, SSL, caching
- **Gray Cloud (DNS-only)**: Stratum mining requires direct connection, can't proxy through Cloudflare

**Alternative (recommended):** Use wildcard DNS:
```
A     *.nicksminingpool.com        →  108.249.26.52  [⚫ DNS-only]
A     nicksminingpool.com          →  108.249.26.52  [🟠 Proxied]
A     www.nicksminingpool.com      →  108.249.26.52  [🟠 Proxied]
```

---

## ✅ Setup Complete!

Your mining pool is now accessible:

### Mining Connections:
```
Bitcoin (BTC):     stratum+tcp://btc.nicksminingpool.com:6004
Bitcoin Cash (BCH): stratum+tcp://bch.nicksminingpool.com:6002
Bitcoin SV (BSV):  stratum+tcp://bsv.nicksminingpool.com:6005
Dogecoin (DOGE):   stratum+tcp://doge.nicksminingpool.com:6003
eCash (XEC):       stratum+tcp://ecash.nicksminingpool.com:6007
```

### Web Interface:
```
https://nicksminingpool.com
```

---

## Common Commands

```bash
# View NPM logs
cd ~/nginx-proxy-manager
docker-compose logs -f app

# Restart NPM
docker-compose restart

# Stop NPM
docker-compose down

# Start NPM
docker-compose up -d

# Update NPM
docker-compose pull
docker-compose up -d

# Check container status
docker-compose ps
```

---

## Troubleshooting

### Can't access admin panel on port 81?

```bash
# Check if container is running
docker ps | grep nginx-proxy-manager

# Check firewall
sudo ufw status | grep 81

# View logs
docker logs nginx-proxy-manager --tail 50
```

### Stratum port not working?

```bash
# Test backend connection from NPM server
nc -zv 172.16.111.20 6004

# Check if port is open
sudo ufw status | grep 6004

# View nginx error log
docker exec nginx-proxy-manager tail -f /var/log/nginx/error.log
```

### SSL certificate failed?

```bash
# Verify DNS points to your server
dig nicksminingpool.com +short

# Ensure port 80 is open (required for Let's Encrypt)
sudo ufw allow 80/tcp

# Check NPM logs
docker logs nginx-proxy-manager | grep -i certbot
```

---

## Security Recommendations

### Restrict Admin Access

```bash
# Only allow admin panel from your IP
sudo ufw delete allow 81/tcp
sudo ufw allow from YOUR_IP_ADDRESS to any port 81 proto tcp
```

### Regular Updates

```bash
# Update NPM monthly
cd ~/nginx-proxy-manager
docker-compose pull
docker-compose up -d

# Update system
sudo apt update && sudo apt upgrade -y
```

### Backup Configuration

```bash
# Backup NPM data
cd ~/nginx-proxy-manager
tar -czf npm-backup-$(date +%Y%m%d).tar.gz data/ letsencrypt/

# Copy backup offsite
scp npm-backup-*.tar.gz user@backup-server:/backups/
```

---

## Need Help?

Check the full documentation:
- `NPM-CONFIGURATION.md` - Complete setup guide
- `MULTI-POOL-DNS-CONFIGURATION.md` - DNS configuration options
- `STRATUM-NGINX-PROXY-MANAGER-GUIDE.md` - Comprehensive stratum guide

---

**Setup Time:** 10-15 minutes
**Difficulty:** Easy
**Last Updated:** January 21, 2026
