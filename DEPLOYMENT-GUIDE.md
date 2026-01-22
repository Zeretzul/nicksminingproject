# Nick's Mining Pool - Deployment Guide

## Overview

Complete step-by-step deployment guide for Nick's Mining Pool with correct network configuration.

**Network:**
- External IP: 108.249.26.52 (Cloudflare)
- NPM Server: 172.16.0.21 (internal)
- Pool Backend: 172.16.111.20 (internal)

**Deployment Time:** 2-3 hours
**Difficulty:** Medium

---

## Pre-Deployment Checklist

Before starting, ensure you have:

- [ ] Ubuntu 20.04/22.04 or Debian 11/12 installed on NPM server (172.16.0.21)
- [ ] Mining pool software running on backend server (172.16.111.20)
- [ ] Root/sudo access to both servers
- [ ] Domain: nicksminingpool.com registered
- [ ] Cloudflare account created
- [ ] Static external IP: 108.249.26.52
- [ ] Router/firewall admin access
- [ ] SSH access to both servers

---

## Phase 1: Router/Firewall Configuration

### Step 1.1: Configure Port Forwarding

Log into your router/firewall admin interface.

**Create port forwarding rules:**

Forward from `108.249.26.52` (WAN) to `172.16.0.21` (NPM Server):

```
External  Protocol  Internal IP     Internal Port  Description
──────────────────────────────────────────────────────────────
80        TCP       172.16.0.21     80            HTTP
443       TCP       172.16.0.21     443           HTTPS
6002      TCP       172.16.0.21     6002          BCH Stratum
6003      TCP       172.16.0.21     6003          DOGE Stratum
6004      TCP       172.16.0.21     6004          BTC Stratum
6005      TCP       172.16.0.21     6005          BSV Stratum
6007      TCP       172.16.0.21     6007          XEC Stratum
```

**Do NOT forward port 81** (NPM admin - internal access only!)

---

### Step 1.2: Verify Port Forwarding

```bash
# From an external network (not your LAN), test each port:
nc -zv 108.249.26.52 80
nc -zv 108.249.26.52 443
nc -zv 108.249.26.52 6004
```

---

## Phase 2: Cloud flare DNS Configuration

### Step 2.1: Add Domain to Cloudflare

1. Visit https://dash.cloudflare.com
2. Click "Add a Site"
3. Enter: `nicksminingpool.com`
4. Select Free plan
5. Note the nameservers provided

### Step 2.2: Update Domain Registrar

At your domain registrar, change nameservers to:
- `sue.ns.cloudflare.com` (example)
- `tim.ns.cloudflare.com` (example)

**Wait:** 5-30 minutes for propagation

### Step 2.3: Add DNS Records

In Cloudflare DNS tab:

```
Type  Name      Content          Proxy Status
──────────────────────────────────────────────
A     @         108.249.26.52    🟠 Proxied
A     www       108.249.26.52    🟠 Proxied
A     btc       108.249.26.52    ⚫ DNS-only
A     bch       108.249.26.52    ⚫ DNS-only
A     bsv       108.249.26.52    ⚫ DNS-only
A     doge      108.249.26.52    ⚫ DNS-only
A     ecash     108.249.26.52    ⚫ DNS-only
```

**Alternative (wildcard):**
```
A     *         108.249.26.52    ⚫ DNS-only
A     @         108.249.26.52    🟠 Proxied
A     www       108.249.26.52    🟠 Proxied
```

### Step 2.4: Configure SSL/TLS

Cloudflare → SSL/TLS → Overview:
- Set to: **Full (strict)**

Cloudflare → SSL/TLS → Edge Certificates:
- Always Use HTTPS: **On**

---

## Phase 3: NPM Server Setup (172.16.0.21)

### Step 3.1: Install Docker

```bash
# SSH into NPM server
ssh admin@172.16.0.21

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add user to docker group
sudo usermod -aG docker $USER

# Logout and login for changes to take effect
exit
ssh admin@172.16.0.21

# Verify
docker --version
docker-compose --version
```

---

### Step 3.2: Deploy NPM

```bash
# Create directory
mkdir -p ~/nginx-proxy-manager
cd ~/nginx-proxy-manager

# Create docker-compose.yml
nano docker-compose.yml
```

**Paste this content:**
```yaml
version: '3.8'

services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '81:81'
      - '80:80'
      - '443:443'
      - '6002:6002'
      - '6003:6003'
      - '6004:6004'
      - '6005:6005'
      - '6007:6007'
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
```

**Save (Ctrl+X, Y, Enter)**

```bash
# Start NPM
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f app
```

---

### Step 3.3: Configure Firewall

```bash
# Allow required ports
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 6002/tcp
sudo ufw allow 6003/tcp
sudo ufw allow 6004/tcp
sudo ufw allow 6005/tcp
sudo ufw allow 6007/tcp

# Allow NPM admin from internal network only
sudo ufw allow from 172.16.0.0/16 to any port 81 proto tcp

# Enable firewall
sudo ufw enable

# Verify
sudo ufw status numbered
```

---

## Phase 4: NPM Configuration

### Step 4.1: Access NPM Admin

1. Open browser to: `http://172.16.0.21:81`
2. Default login:
   - Email: `admin@example.com`
   - Password: `changeme`
3. **IMMEDIATELY** change password after first login!

---

### Step 4.2: Configure Streams (Mining Ports)

For each coin, add a TCP stream:

**Bitcoin (BTC):**
1. Click **Streams** → **Add Stream**
2. Incoming Port: `6004`
3. Forwarding Host: `172.16.111.20`
4. Forwarding Port: `6004`
5. TCP Forwarding: ✅ Enabled
6. UDP Forwarding: ❌ Disabled
7. Click **Save**

**Repeat for:**
- BCH: Port `6002` → `172.16.111.20:6002`
- BSV: Port `6005` → `172.16.111.20:6005`
- DOGE: Port `6003` → `172.16.111.20:6003`
- XEC: Port `6007` → `172.16.111.20:6007`

---

### Step 4.3: Configure Proxy Host (Web Interface)

1. Click **Hosts** → **Proxy Hosts** → **Add Proxy Host**
2. **Details tab:**
   - Domain Names: `nicksminingpool.com`, `www.nicksminingpool.com`
   - Scheme: `http`
   - Forward Hostname/IP: `172.16.111.20`
   - Forward Port: `8559`
   - Block Common Exploits: ✅
   - Websockets Support: ✅
3. **SSL tab:**
   - SSL Certificate: **Request a new SSL Certificate**
   - Force SSL: ✅
   - HTTP/2 Support: ✅
   - Email: your-email@example.com
   - I Agree to Let's Encrypt Terms: ✅
4. Click **Save**

NPM will automatically request and configure SSL certificate.

---

## Phase 5: Verify Deployment

### Test 1: Web Interface

```bash
# Should redirect to HTTPS
curl -I http://nicksminingpool.com

# Should return 200 OK
curl -I https://nicksminingpool.com
```

**Browser test:**
- Visit: https://nicksminingpool.com
- Should show pool website with valid SSL (green padlock)

---

### Test 2: Stratum Connections

```bash
# Test each mining port
telnet btc.nicksminingpool.com 6004
telnet bch.nicksminingpool.com 6002
telnet bsv.nicksminingpool.com 6005
telnet doge.nicksminingpool.com 6003
telnet ecash.nicksminingpool.com 6007
```

**Expected:** Connection established for each port

**Send test command:**
```bash
echo '{"id": 1, "method": "mining.subscribe", "params": []}' | nc btc.nicksminingpool.com 6004
```

**Expected:** JSON response from pool

---

### Test 3: DNS Resolution

```bash
# Main domain (should return Cloudflare IP)
dig nicksminingpool.com +short
# Example: 104.21.x.x

# Stratum domains (should return YOUR IP)
dig btc.nicksminingpool.com +short
# Expected: 108.249.26.52
```

---

### Test 4: SSL Certificate

```bash
# Check certificate
echo | openssl s_client -connect nicksminingpool.com:443 -servername nicksminingpool.com 2>/dev/null | grep -i "issuer"

# Check expiry
echo | openssl s_client -connect nicksminingpool.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## Phase 6: Post-Deployment

### Step 6.1: Secure NPM Admin Access

**Option A: VPN Access Only**
```bash
# Block external access to port 81
sudo ufw delete allow from any to any port 81
sudo ufw allow from VPN_SUBNET to any port 81 proto tcp
```

**Option B: Specific IP Only**
```bash
# Allow only from your admin IP
sudo ufw delete allow 81/tcp
sudo ufw allow from YOUR_ADMIN_IP to any port 81 proto tcp
```

---

### Step 6.2: Set Up Monitoring

```bash
# Monitor NPM logs
docker logs -f nginx-proxy-manager

# Monitor connection count
watch 'netstat -an | grep -E "(6002|6003|6004|6005|6007)" | grep ESTABLISHED | wc -l'

# Monitor resource usage
docker stats nginx-proxy-manager
```

---

### Step 6.3: Configure Backups

```bash
# Create backup script
nano ~/backup-npm.sh
```

**Content:**
```bash
#!/bin/bash
BACKUP_DIR="/backup/npm-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

cd ~/nginx-proxy-manager
tar -czf $BACKUP_DIR/npm-config.tar.gz data/ letsencrypt/

# Keep last 30 days
find /backup/ -type d -mtime +30 -exec rm -rf {} \;

echo "Backup completed: $(date)" >> ~/backup.log
```

**Make executable and schedule:**
```bash
chmod +x ~/backup-npm.sh

# Add to crontab (daily at 3 AM)
crontab -e
# Add line:
0 3 * * * /home/yourusername/backup-npm.sh
```

---

### Step 6.4: Document Configuration

Create a configuration record:

```bash
nano ~/npm-config-notes.txt
```

**Content:**
```
Nick's Mining Pool - NPM Configuration
Deployment Date: [DATE]

External IP: 108.249.26.52
NPM Server: 172.16.0.21
Pool Backend: 172.16.111.20

Port Mapping:
- 80/443 → Web Interface
- 6002 → BCH
- 6003 → DOGE
- 6004 → BTC
- 6005 → BSV
- 6007 → XEC

Cloudflare:
- Web: Proxied (🟠)
- Stratum: DNS-only (⚫)

SSL Certificate: Let's Encrypt
Auto-renewal: Enabled

NPM Admin: http://172.16.0.21:81
Username: admin@example.com
Password: [CHANGED on YYYY-MM-DD]

Backup Location: /backup/
Backup Schedule: Daily 3 AM
```

---

## Troubleshooting

### Issue: NPM Won't Start

```bash
# Check Docker
sudo systemctl status docker

# Check logs
docker logs nginx-proxy-manager --tail 100

# Restart
cd ~/nginx-proxy-manager
docker-compose restart
```

---

### Issue: Can't Access Web Interface

```bash
# Check if proxied correctly
curl -I https://nicksminingpool.com

# Test backend directly from NPM server
curl http://172.16.111.20:8559

# Check NPM proxy host configuration
# In NPM GUI: Hosts → Proxy Hosts → Edit
```

---

### Issue: Miners Can't Connect

```bash
# Verify DNS (should return YOUR IP, not Cloudflare)
dig btc.nicksminingpool.com +short

# Test direct connection
telnet btc.nicksminingpool.com 6004

# Check stream configuration
docker exec nginx-proxy-manager cat /data/nginx/stream/*.conf

# Check backend is listening
nc -zv 172.16.111.20 6004
```

---

## Rollback Plan

If deployment fails:

1. **Stop NPM:**
   ```bash
   cd ~/nginx-proxy-manager
   docker-compose down
   ```

2. **Restore from backup:**
   ```bash
   rm -rf data/ letsencrypt/
   tar -xzf /backup/npm-YYYYMMDD/npm-config.tar.gz
   ```

3. **Restart:**
   ```bash
   docker-compose up -d
   ```

4. **Verify:**
   - Check NPM admin access
   - Test web interface
   - Test stratum connections

---

## Maintenance Schedule

### Daily
- Check NPM logs for errors
- Monitor connection count
- Verify all services running

### Weekly
- Review Cloudflare analytics
- Check SSL certificate expiry
- Review firewall logs

### Monthly
- Update NPM Docker image
- Test backup restoration
- Review and update documentation
- Audit security settings

---

## Next Steps After Deployment

1. ✅ **Verify all services working**
2. ✅ **Configure monitoring alerts**
3. ✅ **Test backup/restore procedure**
4. ✅ **Document any custom changes**
5. ✅ **Create runbook for common issues**
6. ⏳ **Plan for SSL stratum (future)**
7. ⏳ **Plan for load balancing (if needed)**

---

**Deployment Complete!** ✅

Your mining pool is now live at:
- **Web**: https://nicksminingpool.com
- **BTC**: stratum+tcp://btc.nicksminingpool.com:6004
- **BCH**: stratum+tcp://bch.nicksminingpool.com:6002
- **BSV**: stratum+tcp://bsv.nicksminingpool.com:6005
- **DOGE**: stratum+tcp://doge.nicksminingpool.com:6003
- **XEC**: stratum+tcp://ecash.nicksminingpool.com:6007

---

**Last Updated:** January 21, 2026
**Guide Version:** 1.0
**For:** Nick's Mining Pool
