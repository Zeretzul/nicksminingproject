# Multiple Mining Pools DNS Configuration Guide

## Quick Answer

**No, you don't *need* multiple subdomains, but YES, it's highly recommended.**

You can run multiple pools on the same domain using different ports, but using subdomains provides better organization, security, and user experience.

---

## Option 1: Single Domain with Multiple Ports (NOT Recommended)

### Configuration Example

```
pool.yourdomain.com:3333  → Bitcoin pool
pool.yourdomain.com:3334  → Bitcoin SSL
pool.yourdomain.com:8008  → Ethereum pool
pool.yourdomain.com:8009  → Ethereum SSL
pool.yourdomain.com:9933  → Litecoin pool
pool.yourdomain.com:9934  → Litecoin SSL
```

### DNS Configuration

```
A     pool.yourdomain.com    →  123.45.67.89
```

### NPM Configuration (docker-compose.yml)

```yaml
services:
  app:
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
      - '3333:3333'   # Bitcoin
      - '3334:3334'   # Bitcoin SSL
      - '8008:8008'   # Ethereum
      - '8009:8009'   # Ethereum SSL
      - '9933:9933'   # Litecoin
      - '9934:9934'   # Litecoin SSL
```

### SSL Certificate Configuration

**Challenge:** Single certificate covers all pools

```bash
# Request certificate for single domain
certbot certonly --standalone -d pool.yourdomain.com
```

**stratum.conf:**
```nginx
# Bitcoin SSL
stream {
    server {
        listen 3334 ssl;
        proxy_pass bitcoin_backend;
        ssl_certificate /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/pool.yourdomain.com/privkey.pem;
    }
}

# Ethereum SSL
stream {
    server {
        listen 8009 ssl;
        proxy_pass ethereum_backend;
        ssl_certificate /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/pool.yourdomain.com/privkey.pem;
    }
}

# Litecoin SSL
stream {
    server {
        listen 9934 ssl;
        proxy_pass litecoin_backend;
        ssl_certificate /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/pool.yourdomain.com/privkey.pem;
    }
}
```

### ❌ Disadvantages

1. **Certificate validation issues**: Miners may have trouble verifying which pool they're connecting to
2. **Port confusion**: Users need to remember different port numbers
3. **No isolation**: All pools share same reputation/DNS record
4. **Certificate revocation impact**: If one pool is compromised, all pools affected
5. **Harder to manage**: Difficult to route different pools to different servers later
6. **Firewall complexity**: Need to open many ports individually

---

## Option 2: Multiple Subdomains (RECOMMENDED ✅)

### Configuration Example

```
btc.yourdomain.com:3333   → Bitcoin pool
btc.yourdomain.com:3334   → Bitcoin SSL

eth.yourdomain.com:3333   → Ethereum pool
eth.yourdomain.com:3334   → Ethereum SSL

ltc.yourdomain.com:3333   → Litecoin pool
ltc.yourdomain.com:3334   → Litecoin SSL
```

### DNS Configuration

```
A     btc.yourdomain.com    →  123.45.67.89
A     eth.yourdomain.com    →  123.45.67.89
A     ltc.yourdomain.com    →  123.45.67.89
A     rvn.yourdomain.com    →  123.45.67.89
A     xmr.yourdomain.com    →  123.45.67.89
```

**OR use a wildcard (easier):**

```
A     *.yourdomain.com      →  123.45.67.89
```

### NPM Configuration (docker-compose.yml)

**Option 2A: All pools on same ports (requires IP aliasing)**

```yaml
services:
  app:
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
      - '3333:3333'   # Standard stratum
      - '3334:3334'   # SSL stratum
    # All coins use same ports, routed by subdomain
```

**Option 2B: Different ports per pool (simpler)**

```yaml
services:
  app:
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
      # Bitcoin
      - '3333:3333'
      - '3334:3334'
      # Ethereum
      - '8008:8008'
      - '8009:8009'
      # Litecoin
      - '9933:9933'
      - '9934:9934'
```

### SSL Certificate Configuration

**Method A: Individual Certificates**

```bash
# Request separate certificate for each subdomain
certbot certonly --standalone -d btc.yourdomain.com
certbot certonly --standalone -d eth.yourdomain.com
certbot certonly --standalone -d ltc.yourdomain.com
```

**Method B: Multi-domain Certificate (SAN)**

```bash
# Single certificate covering all subdomains
certbot certonly --standalone \
  -d btc.yourdomain.com \
  -d eth.yourdomain.com \
  -d ltc.yourdomain.com \
  -d rvn.yourdomain.com
```

**Method C: Wildcard Certificate (BEST ✅)**

```bash
# One certificate for *.yourdomain.com (requires DNS challenge)
certbot certonly --manual \
  --preferred-challenges dns \
  -d "*.yourdomain.com"
```

**You'll need to add TXT record:**
```
TXT   _acme-challenge.yourdomain.com    →  [certbot-provided-value]
```

### stratum.conf (Same ports, different subdomains)

```nginx
# Bitcoin Pool
stream {
    upstream bitcoin_backend {
        server 127.0.0.1:13333;  # Internal Bitcoin pool
    }

    # Bitcoin non-SSL
    server {
        listen 3333;
        server_name btc.yourdomain.com;
        proxy_pass bitcoin_backend;
        proxy_timeout 300s;
        proxy_socket_keepalive on;
    }

    # Bitcoin SSL
    server {
        listen 3334 ssl;
        server_name btc.yourdomain.com;
        proxy_pass bitcoin_backend;
        proxy_timeout 300s;

        ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
    }
}

# Ethereum Pool
stream {
    upstream ethereum_backend {
        server 127.0.0.1:18008;  # Internal Ethereum pool
    }

    # Ethereum non-SSL
    server {
        listen 3333;
        server_name eth.yourdomain.com;
        proxy_pass ethereum_backend;
        proxy_timeout 300s;
    }

    # Ethereum SSL
    server {
        listen 3334 ssl;
        server_name eth.yourdomain.com;
        proxy_pass ethereum_backend;
        proxy_timeout 300s;

        ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
    }
}

# Litecoin Pool
stream {
    upstream litecoin_backend {
        server 127.0.0.1:19933;  # Internal Litecoin pool
    }

    # Litecoin non-SSL
    server {
        listen 3333;
        server_name ltc.yourdomain.com;
        proxy_pass litecoin_backend;
        proxy_timeout 300s;
    }

    # Litecoin SSL
    server {
        listen 3334 ssl;
        server_name ltc.yourdomain.com;
        proxy_pass litecoin_backend;
        proxy_timeout 300s;

        ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
    }
}
```

**Note:** Nginx stream module doesn't natively support SNI (Server Name Indication) routing for TCP streams in open-source version. The `server_name` directive in stream context requires Nginx Plus or requires using different ports.

### ✅ Advantages

1. **Clear organization**: Each coin has its own subdomain
2. **Flexible routing**: Can move pools to different servers later
3. **Better SSL management**: Wildcard certificate covers all pools
4. **Professional appearance**: `btc.pool.com` vs `pool.com:3333`
5. **Easier troubleshooting**: Can isolate issues per pool
6. **Independent monitoring**: Set up separate monitoring per subdomain
7. **SEO and branding**: Each pool can have its own web interface
8. **Firewall rules**: Easier to manage per-subdomain rules

---

## Option 3: Hybrid Approach (Best for Production)

### Configuration

Use subdomains for pool categories, ports for SSL/non-SSL:

```
# Bitcoin family (SHA-256)
sha256.yourdomain.com:3333    → Bitcoin non-SSL
sha256.yourdomain.com:3334    → Bitcoin SSL
sha256.yourdomain.com:3335    → Bitcoin Cash non-SSL
sha256.yourdomain.com:3336    → Bitcoin Cash SSL

# Ethereum family (Ethash)
ethash.yourdomain.com:3333    → Ethereum non-SSL
ethash.yourdomain.com:3334    → Ethereum SSL
ethash.yourdomain.com:3335    → Ethereum Classic non-SSL
ethash.yourdomain.com:3336    → Ethereum Classic SSL

# Scrypt family
scrypt.yourdomain.com:3333    → Litecoin non-SSL
scrypt.yourdomain.com:3334    → Litecoin SSL
scrypt.yourdomain.com:3335    → Dogecoin non-SSL
scrypt.yourdomain.com:3336    → Dogecoin SSL
```

### DNS Configuration

```
A     sha256.yourdomain.com    →  123.45.67.89
A     ethash.yourdomain.com    →  123.45.67.89
A     scrypt.yourdomain.com    →  123.45.67.89
A     equihash.yourdomain.com  →  123.45.67.89
```

### ✅ Advantages

1. **Algorithmic organization**: Group coins by mining algorithm
2. **Reduced DNS records**: Fewer subdomains to manage
3. **Shared infrastructure**: Algorithm families can share backend resources
4. **Easier pool switching**: Miners can switch coins without changing domain

---

## Option 4: Geographic Distribution

For large-scale operations with multiple servers:

### Configuration

```
# US East Coast
us-east.yourdomain.com:3333   → BTC (Virginia server)
us-east.yourdomain.com:8008   → ETH (Virginia server)

# US West Coast
us-west.yourdomain.com:3333   → BTC (California server)
us-west.yourdomain.com:8008   → ETH (California server)

# Europe
eu.yourdomain.com:3333        → BTC (Frankfurt server)
eu.yourdomain.com:8008        → ETH (Frankfurt server)

# Asia
asia.yourdomain.com:3333      → BTC (Singapore server)
asia.yourdomain.com:8008      → ETH (Singapore server)
```

### DNS Configuration with GeoDNS

```
# Route users to nearest server
A     pool.yourdomain.com    →  Geo-routed based on miner IP
```

### Use GeoDNS Services:
- **Cloudflare**: Load balancing + geo-routing
- **AWS Route 53**: Geolocation routing
- **NS1**: Geographic routing

---

## Recommended Architecture for Your Pool

### Small Pool (1-5 coins, single server)

**DNS Setup:**
```
A     btc.pool.yourdomain.com    →  123.45.67.89
A     eth.pool.yourdomain.com    →  123.45.67.89
A     ltc.pool.yourdomain.com    →  123.45.67.89
```

**SSL:** Wildcard certificate `*.pool.yourdomain.com`

**Ports:** Standardize to 3333/3334 for all pools

**Total DNS records:** 3-5

---

### Medium Pool (5-15 coins, 1-3 servers)

**DNS Setup:**
```
# Main pool interface
A     pool.yourdomain.com        →  123.45.67.89

# Wildcard for coin-specific subdomains
A     *.yourdomain.com           →  123.45.67.89

# Specific coins
A     btc.yourdomain.com         →  123.45.67.89
A     eth.yourdomain.com         →  123.45.67.89
A     ltc.yourdomain.com         →  123.45.67.89
A     xmr.yourdomain.com         →  123.45.67.89
A     rvn.yourdomain.com         →  123.45.67.89
```

**SSL:** Wildcard certificate `*.yourdomain.com`

**Ports:**
- Standard: 3333/3334
- Or unique per coin if using different ports

**Total DNS records:** 1 wildcard + 5-15 specific

---

### Large Pool (15+ coins, multiple servers)

**DNS Setup:**
```
# Main website
A     www.yourdomain.com         →  Web server IP

# Wildcard for all pool subdomains
A     *.pool.yourdomain.com      →  Main pool server
A     *.us.yourdomain.com        →  US pool server
A     *.eu.yourdomain.com        →  EU pool server
A     *.asia.yourdomain.com      →  Asia pool server

# Specific coin pools with geo-routing
A     btc.yourdomain.com         →  GeoDNS routing
A     eth.yourdomain.com         →  GeoDNS routing
```

**SSL:** Multiple wildcard certificates

**Ports:** Standardized across all servers

**Load balancing:** Use Cloudflare or AWS Route 53

---

## SSL Certificate Decision Matrix

| Scenario | SSL Solution | Cost | Complexity | Renewals |
|----------|-------------|------|------------|----------|
| **Single domain, multiple ports** | Single cert for `pool.domain.com` | Free | Low | 1 renewal |
| **2-3 subdomains** | Individual certs per subdomain | Free | Medium | 2-3 renewals |
| **4+ subdomains** | Multi-domain (SAN) certificate | Free | Medium | 1 renewal |
| **Many subdomains (5+)** | Wildcard `*.domain.com` | Free* | Medium | 1 renewal |
| **Multi-level subdomains** | Wildcard `*.pool.domain.com` | Free* | High | 1 renewal |
| **Enterprise (100+ domains)** | Commercial wildcard cert | Paid | Low | 1 renewal |

*Requires DNS challenge for Let's Encrypt wildcard certificates

---

## Step-by-Step: Setting Up Wildcard Certificate

### Step 1: Choose DNS Provider with API Support

Certbot supports automated DNS validation for these providers:
- Cloudflare (recommended)
- AWS Route 53
- Google Cloud DNS
- DigitalOcean DNS
- Many others

### Step 2: Install Certbot DNS Plugin

**For Cloudflare:**
```bash
# Install Cloudflare DNS plugin
sudo apt install -y python3-certbot-dns-cloudflare

# Create Cloudflare credentials file
mkdir -p ~/.secrets/certbot
nano ~/.secrets/certbot/cloudflare.ini
```

**cloudflare.ini:**
```ini
dns_cloudflare_api_token = your_cloudflare_api_token_here
```

```bash
# Secure the file
chmod 600 ~/.secrets/certbot/cloudflare.ini
```

### Step 3: Request Wildcard Certificate

```bash
# Request wildcard cert
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini \
  -d "*.yourdomain.com" \
  -d "yourdomain.com"
```

**Output:**
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/yourdomain.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/yourdomain.com/privkey.pem
```

### Step 4: Copy Certificate to NPM Container

```bash
# Copy wildcard cert to NPM
docker cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
  nginx-proxy-manager:/etc/letsencrypt/live/yourdomain.com/fullchain.pem

docker cp /etc/letsencrypt/live/yourdomain.com/privkey.pem \
  nginx-proxy-manager:/etc/letsencrypt/live/yourdomain.com/privkey.pem

# Or mount Let's Encrypt directory in docker-compose.yml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
```

### Step 5: Configure Auto-Renewal

```bash
# Create renewal hook script
sudo nano /etc/letsencrypt/renewal-hooks/deploy/restart-npm.sh
```

```bash
#!/bin/bash
docker exec nginx-proxy-manager nginx -s reload
```

```bash
# Make executable
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/restart-npm.sh

# Test renewal
sudo certbot renew --dry-run
```

---

## Practical Example: 5-Pool Setup

### Scenario

You want to run:
1. Bitcoin (BTC)
2. Ethereum (ETH)
3. Litecoin (LTC)
4. Ravencoin (RVN)
5. Monero (XMR)

### Recommended Configuration

**DNS Records:**
```
# Wildcard (covers all coin subdomains)
A     *.pool.yourdomain.com    →  123.45.67.89

# Explicit records (optional, for clarity)
A     btc.pool.yourdomain.com  →  123.45.67.89
A     eth.pool.yourdomain.com  →  123.45.67.89
A     ltc.pool.yourdomain.com  →  123.45.67.89
A     rvn.pool.yourdomain.com  →  123.45.67.89
A     xmr.pool.yourdomain.com  →  123.45.67.89

# Main website
A     www.yourdomain.com       →  123.45.67.89
```

**SSL Certificate:**
```bash
# Single wildcard certificate
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini \
  -d "*.pool.yourdomain.com" \
  -d "pool.yourdomain.com"
```

**Port Standardization:**
```
All pools use:
- Port 3333 for non-SSL stratum
- Port 3334 for SSL stratum
- Port 8080 for pool website/API (behind NPM reverse proxy)
```

**NPM docker-compose.yml:**
```yaml
version: '3.8'

services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # Admin UI
      - '81:81'
      # HTTP/HTTPS (for web interfaces)
      - '80:80'
      - '443:443'
      # Stratum (standardized)
      - '3333:3333'
      - '3334:3334'
    volumes:
      - ./data:/data
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - db
```

**Backend Pool Configuration:**
```
Bitcoin pool:    Listening on 127.0.0.1:13333
Ethereum pool:   Listening on 127.0.0.1:13334
Litecoin pool:   Listening on 127.0.0.1:13335
Ravencoin pool:  Listening on 127.0.0.1:13336
Monero pool:     Listening on 127.0.0.1:13337
```

**stratum.conf:**
```nginx
# Map subdomain to backend
map $ssl_preread_server_name $backend {
    btc.pool.yourdomain.com  127.0.0.1:13333;
    eth.pool.yourdomain.com  127.0.0.1:13334;
    ltc.pool.yourdomain.com  127.0.0.1:13335;
    rvn.pool.yourdomain.com  127.0.0.1:13336;
    xmr.pool.yourdomain.com  127.0.0.1:13337;
    default                   127.0.0.1:13333;
}

# Non-SSL stratum (port 3333)
server {
    listen 3333;
    proxy_pass $backend;
    proxy_timeout 300s;
    ssl_preread on;
}

# SSL stratum (port 3334)
server {
    listen 3334 ssl;
    proxy_pass $backend;
    proxy_timeout 300s;

    ssl_certificate /etc/letsencrypt/live/pool.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pool.yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_preread on;
}
```

**Miner Connection Examples:**
```bash
# Bitcoin
cgminer -o stratum+tcp://btc.pool.yourdomain.com:3333 -u WALLET.worker1 -p x

# Ethereum
ethminer -P stratum+ssl://WALLET@eth.pool.yourdomain.com:3334

# Litecoin
cpuminer -o stratum+tcp://ltc.pool.yourdomain.com:3333 -u WALLET.worker1 -p x

# Ravencoin
kawpowminer -P stratum+tcp://WALLET@rvn.pool.yourdomain.com:3333

# Monero
xmrig -o xmr.pool.yourdomain.com:3333 -u WALLET -k
```

---

## Cost Comparison

### Single Domain Setup
- **DNS records:** 1 A record
- **SSL certificates:** 1 certificate
- **Annual cost:** $0 (Let's Encrypt free)
- **Management time:** 1 hour/month

### Multi-Subdomain Setup (5 coins)
- **DNS records:** 5 A records (or 1 wildcard)
- **SSL certificates:** 1 wildcard certificate
- **Annual cost:** $0 (Let's Encrypt free)
- **Management time:** 2 hours/month

### Enterprise Setup (20+ coins, geo-distributed)
- **DNS records:** 20+ A records + GeoDNS
- **SSL certificates:** 1-3 wildcard certificates
- **Annual cost:**
  - GeoDNS service: $20-200/month (Cloudflare, Route 53)
  - Commercial SSL (optional): $150-300/year
- **Management time:** 10+ hours/month

---

## My Recommendation

### ✅ **Use Multiple Subdomains with Wildcard SSL**

**Why:**
1. **Professional**: Clean URLs like `btc.pool.yourdomain.com:3333`
2. **Flexible**: Easy to add new coins (just update DNS)
3. **Manageable**: Single wildcard SSL certificate
4. **Scalable**: Can migrate pools to different servers later
5. **Standard**: Industry best practice for multi-coin pools

**Setup Time:** ~2 hours
**Ongoing Maintenance:** ~30 minutes/month
**Cost:** $0 (using Let's Encrypt and free DNS)

---

## Quick Setup Commands

```bash
# 1. Set up DNS (via your DNS provider web interface)
# Add wildcard: *.pool.yourdomain.com → YOUR_SERVER_IP

# 2. Request wildcard SSL certificate
sudo apt install -y python3-certbot-dns-cloudflare
nano ~/.secrets/certbot/cloudflare.ini  # Add API token

sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini \
  -d "*.pool.yourdomain.com" \
  -d "pool.yourdomain.com"

# 3. Update NPM docker-compose.yml
# Mount certificates:
# volumes:
#   - /etc/letsencrypt:/etc/letsencrypt:ro

# 4. Configure stratum.conf with subdomain routing
# (See example above)

# 5. Test connections
telnet btc.pool.yourdomain.com 3333
telnet eth.pool.yourdomain.com 3333
telnet ltc.pool.yourdomain.com 3333

# Done! ✅
```

---

## Cloudflare Configuration for Mining Pools

### Important: Proxy Settings for Stratum

⚠️ **CRITICAL**: Mining pools require special Cloudflare configuration!

**The Problem:**
- Cloudflare's proxy (orange cloud) is designed for HTTP/HTTPS traffic
- Stratum mining protocol uses raw TCP connections
- Proxying stratum traffic through Cloudflare will **BREAK** mining connections

**The Solution:**
Use different proxy settings for web vs mining:

### Cloudflare Proxy Settings

#### 🟠 Orange Cloud (Proxied) - Use for Web Interface Only

**Enable for:**
- `pool.yourdomain.com` (main website)
- `www.pool.yourdomain.com` (main website)

**Benefits:**
- DDoS protection
- SSL/TLS encryption
- Caching for faster page loads
- Web Application Firewall (WAF)
- Analytics

**How to enable:**
- In Cloudflare DNS, click the cloud icon until it's orange

---

#### ⚫ Gray Cloud (DNS-only) - Use for All Mining Subdomains

**Enable for:**
- `btc.pool.yourdomain.com` (stratum port)
- `bch.pool.yourdomain.com` (stratum port)
- `bsv.pool.yourdomain.com` (stratum port)
- `doge.pool.yourdomain.com` (stratum port)
- `ecash.pool.yourdomain.com` (stratum port)
- Any other coin-specific subdomain

**Why DNS-only:**
- Miners need direct TCP connection to your server
- Stratum protocol doesn't work through HTTP proxy
- Lower latency for mining connections
- No Cloudflare limitations on connection count

**How to enable:**
- In Cloudflare DNS, click the cloud icon until it's gray

---

### Practical Cloudflare Setup for Nick's Mining Pool

**Example Configuration:**

```
Cloudflare DNS Records:

A     nicksminingpool.com          →  108.249.26.52  [🟠 Proxied]
A     www.nicksminingpool.com      →  108.249.26.52  [🟠 Proxied]

A     btc.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only]
A     bch.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only]
A     bsv.nicksminingpool.com      →  108.249.26.52  [⚫ DNS-only]
A     doge.nicksminingpool.com     →  108.249.26.52  [⚫ DNS-only]
A     ecash.nicksminingpool.com    →  108.249.26.52  [⚫ DNS-only]
```

**Traffic Flow:**

```
Web Traffic:
User Browser → Cloudflare (orange cloud) → Your Server
- Benefits from DDoS protection, caching, WAF

Mining Traffic:
Miner → Direct to Your Server (gray cloud)
- No Cloudflare proxy, direct TCP connection
```

### Alternative: Wildcard with Manual Overrides

**Best Practice:**

```
A     *.nicksminingpool.com        →  108.249.26.52  [⚫ DNS-only]
A     nicksminingpool.com          →  108.249.26.52  [🟠 Proxied]
A     www.nicksminingpool.com      →  108.249.26.52  [🟠 Proxied]
```

**How this works:**
1. Wildcard `*.nicksminingpool.com` covers all subdomains (DNS-only)
2. Explicit records for main domain override wildcard (Proxied)
3. All coin subdomains automatically use DNS-only
4. Main website gets Cloudflare protection

**Advantages:**
- Don't need to add DNS record for each new coin
- Main website still protected
- All stratum connections work correctly

---

### Cloudflare Firewall Rules (Optional)

**Protect Web Interface while Allowing Mining:**

1. **Go to**: Cloudflare Dashboard → Firewall → Firewall Rules

2. **Create Rule**: "Allow Known Miners"
```
Field: Hostname
Operator: contains
Value: nicksminingpool.com
AND
Field: URI Path
Operator: does not contain
Value: /admin
→ Action: Allow
```

3. **Create Rule**: "Challenge Suspicious Traffic"
```
Field: Hostname
Operator: equals
Value: nicksminingpool.com
AND
Field: Threat Score
Operator: greater than
Value: 10
→ Action: Managed Challenge
```

**Note**: These rules only apply to proxied (orange cloud) domains.

---

### Common Cloudflare Issues & Solutions

#### Issue 1: Miners Can't Connect

**Symptom:**
- Web interface works fine
- Miners get "Connection refused" or timeout

**Solution:**
```
1. Check Cloudflare DNS for mining subdomains
2. Ensure cloud icon is GRAY (not orange)
3. Test: dig btc.nicksminingpool.com +short
   - Should return YOUR external IP (108.249.26.52)
   - NOT a Cloudflare IP (starting with 104.x, 172.x proxy ranges)
```

---

#### Issue 2: SSL Certificate Errors on Web Interface

**Symptom:**
- Certificate mismatch or "Invalid Certificate"

**Solution:**
```
1. In Cloudflare: SSL/TLS → Overview
2. Set SSL mode to "Full" or "Full (strict)"
3. Ensure NPM has valid Let's Encrypt certificate
4. Clear browser cache
```

---

#### Issue 3: Rate Limiting on Mining Connections

**Symptom:**
- Some miners connect, others don't
- Intermittent connection issues

**Solution:**
```
This shouldn't happen with DNS-only (gray cloud)
If it does:
1. Verify cloud is GRAY not orange
2. Check Cloudflare "Security" → "WAF" - ensure rules don't block
3. Consider using different subdomain for problematic connections
```

---

### Cloudflare + NPM Best Practices

**DO:**
- ✅ Use orange cloud for web interface (HTTP/HTTPS)
- ✅ Use gray cloud for all stratum mining ports
- ✅ Use wildcard DNS with explicit overrides
- ✅ Enable "Always Use HTTPS" for main domain
- ✅ Set up Firewall Rules for admin protection

**DON'T:**
- ❌ Proxy stratum traffic through Cloudflare (orange cloud)
- ❌ Use "Development Mode" in production
- ❌ Enable "I'm Under Attack Mode" globally (only for web)
- ❌ Use Cloudflare Workers for mining traffic
- ❌ Enable "Rocket Loader" on pool interface (breaks JavaScript)

---

### Testing Cloudflare Configuration

```bash
# 1. Test DNS resolution
dig nicksminingpool.com +short
# Should return Cloudflare IP (104.x range) if proxied

dig btc.nicksminingpool.com +short
# Should return YOUR external IP (108.249.26.52) if DNS-only

# 2. Test web interface
curl -I https://nicksminingpool.com
# Should see Cloudflare headers (cf-ray, cf-cache-status)

# 3. Test stratum connection
telnet btc.nicksminingpool.com 6004
# Should connect directly to your server

# 4. Verify SSL
echo | openssl s_client -connect nicksminingpool.com:443 -servername nicksminingpool.com 2>/dev/null | grep -i "issuer"
# Should show Cloudflare or Let's Encrypt certificate
```

---

## Summary Table

| Approach | DNS Records | SSL Certs | Complexity | Recommended For |
|----------|-------------|-----------|------------|-----------------|
| **Single domain, multiple ports** | 1 | 1 | Low | Testing only |
| **Multiple subdomains** | 5-10 | 1 wildcard | Medium | **Most pools** ✅ |
| **Algorithm-based subdomains** | 3-5 | 1 wildcard | Medium | Multi-algo pools |
| **Geographic distribution** | 10-20 | 2-3 wildcards | High | Large operations |

**Winner:** Multiple subdomains with wildcard SSL certificate 🏆

---

**Last Updated:** January 21, 2026
