# SSL Stratum Configuration - Future Implementation Guide

## Overview

This guide explains how to add SSL/TLS support to your stratum mining ports in the future.

**Current State:** Non-SSL stratum only (ports 6002-6007)
**Future State:** Both non-SSL and SSL stratum ports

---

## Why Add SSL to Stratum?

### Benefits
- ✅ **Encrypted mining traffic** - Prevents ISP throttling
- ✅ **Privacy** - Hides pool address from ISP
- ✅ **Security** - Prevents man-in-the-middle attacks
- ✅ **Professional** - Shows commitment to miner security

### Drawbacks
- ❌ **Slight latency increase** (usually negligible)
- ❌ **More ports to manage**
- ❌ **Certificate management overhead**

---

## Port Allocation Plan

### Recommended Port Scheme

| Coin | Non-SSL Port | SSL Port | Offset |
|------|--------------|----------|--------|
| Bitcoin Cash (BCH) | 6002 | 6102 | +100 |
| Dogecoin (DOGE) | 6003 | 6103 | +100 |
| Bitcoin (BTC) | 6004 | 6104 | +100 |
| Bitcoin SV (BSV) | 6005 | 6105 | +100 |
| eCash (XEC) | 6007 | 6107 | +100 |

**Pattern:** SSL port = Non-SSL port + 100

---

## Implementation Steps

### Step 1: Update Router Port Forwarding

Add SSL port forwarding rules:

```
External  Protocol  Internal IP     Internal Port
──────────────────────────────────────────────────
6102      TCP       172.16.0.21     6102   # BCH SSL
6103      TCP       172.16.0.21     6103   # DOGE SSL
6104      TCP       172.16.0.21     6104   # BTC SSL
6105      TCP       172.16.0.21     6105   # BSV SSL
6107      TCP       172.16.0.21     6107   # XEC SSL
```

---

### Step 2: Update NPM docker-compose.yml

Edit `~/nginx-proxy-manager/docker-compose.yml`:

```yaml
services:
  app:
    ports:
      # Existing ports
      - '80:80'
      - '443:443'
      - '81:81'
      - '6002:6002'
      - '6003:6003'
      - '6004:6004'
      - '6005:6005'
      - '6007:6007'

      # Add SSL ports
      - '6102:6102'
      - '6103:6103'
      - '6104:6104'
      - '6105:6105'
      - '6107:6107'
```

**Restart NPM:**
```bash
cd ~/nginx-proxy-manager
docker-compose down
docker-compose up -d
```

---

### Step 3: Configure Firewall

```bash
# Allow SSL stratum ports
sudo ufw allow 6102/tcp  # BCH SSL
sudo ufw allow 6103/tcp  # DOGE SSL
sudo ufw allow 6104/tcp  # BTC SSL
sudo ufw allow 6105/tcp  # BSV SSL
sudo ufw allow 6107/tcp  # XEC SSL
```

---

### Step 4: Update stratum-streams.conf

Create SSL stream configurations:

```nginx
# Bitcoin (BTC) SSL - Port 6104
stream {
    upstream btc_ssl_backend {
        server 172.16.111.20:6004 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6104 ssl;
        proxy_pass btc_ssl_backend;
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
        proxy_socket_keepalive on;

        # SSL Configuration
        ssl_certificate /etc/letsencrypt/live/nicksminingpool.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/nicksminingpool.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
    }
}

# Repeat for BCH (6102), DOGE (6103), BSV (6105), XEC (6107)
```

**Apply configuration:**
```bash
docker cp stratum-streams.conf nginx-proxy-manager:/data/nginx/custom/
docker exec nginx-proxy-manager nginx -t
docker exec nginx-proxy-manager nginx -s reload
```

---

### Step 5: Test SSL Connections

```bash
# Test SSL handshake
openssl s_client -connect btc.nicksminingpool.com:6104

# Test with miner
cgminer -o stratum+ssl://btc.nicksminingpool.com:6104 -u WALLET.worker1 -p x
```

---

## Miner Connection Examples

### Bitcoin (BTC)

**Non-SSL:**
```
stratum+tcp://btc.nicksminingpool.com:6004
```

**SSL:**
```
stratum+ssl://btc.nicksminingpool.com:6104
```

### Bitcoin Cash (BCH)

**Non-SSL:**
```
stratum+tcp://bch.nicksminingpool.com:6002
```

**SSL:**
```
stratum+ssl://bch.nicksminingpool.com:6102
```

*Repeat pattern for other coins*

---

## Certificate Management

### Using Existing Let's Encrypt Certificate

The web interface certificate (`/etc/letsencrypt/live/nicksminingpool.com/`) can be reused for stratum SSL.

**Advantages:**
- No additional certificates needed
- Auto-renewal already configured
- Same trust chain as website

---

### Alternative: Dedicated Stratum Certificate

If you want separate certificates for stratum:

```bash
# Request new certificate for stratum subdomains
sudo certbot certonly --standalone \
  -d btc.nicksminingpool.com \
  -d bch.nicksminingpool.com \
  -d bsv.nicksminingpool.com \
  -d doge.nicksminingpool.com \
  -d ecash.nicksminingpool.com
```

**Update stratum.conf to use new cert:**
```nginx
ssl_certificate /etc/letsencrypt/live/btc.nicksminingpool.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/btc.nicksminingpool.com/privkey.pem;
```

---

## Troubleshooting

### Issue: SSL Handshake Fails

```bash
# Test certificate
openssl s_client -connect btc.nicksminingpool.com:6104 -showcerts

# Check certificate exists
docker exec nginx-proxy-manager ls -la /etc/letsencrypt/live/nicksminingpool.com/

# Verify nginx config
docker exec nginx-proxy-manager nginx -t
```

---

### Issue: Miners Can't Connect to SSL Port

```bash
# Test port accessibility
telnet btc.nicksminingpool.com 6104

# Check firewall
sudo ufw status | grep 6104

# Check NPM is listening
docker exec nginx-proxy-manager netstat -tlnp | grep 6104
```

---

## Performance Considerations

### SSL Overhead

**CPU Impact:**
- Negligible on modern CPUs (AES-NI acceleration)
- ~1-2% CPU increase per 100 miners

**Latency Impact:**
- SSL handshake: +10-50ms (one-time)
- Ongoing connection: <1ms overhead

---

## Future Enhancements

### Idea 1: Automatic SSL Redirect

Configure pool to automatically upgrade non-SSL connections:

```
Miner connects to port 6004 (non-SSL)
→ Pool sends "upgrade to SSL" message
→ Miner reconnects to port 6104 (SSL)
```

---

### Idea 2: Client Certificate Authentication

Add extra security layer:

```nginx
ssl_client_certificate /path/to/ca.crt;
ssl_verify_client on;
```

Only miners with valid client certificates can connect.

---

## Complete SSL Configuration Example

**Full stratum-streams.conf with SSL:**

```nginx
# Bitcoin Cash (BCH) - Non-SSL
stream {
    upstream bch_backend {
        server 172.16.111.20:6002;
    }
    server {
        listen 6002;
        proxy_pass bch_backend;
    }
}

# Bitcoin Cash (BCH) - SSL
stream {
    upstream bch_ssl_backend {
        server 172.16.111.20:6002;
    }
    server {
        listen 6102 ssl;
        proxy_pass bch_ssl_backend;
        ssl_certificate /etc/letsencrypt/live/nicksminingpool.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/nicksminingpool.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
    }
}

# Repeat for all other coins...
```

---

**Estimated Implementation Time:** 1-2 hours
**Recommended Timeline:** Implement after pool is stable with non-SSL connections

---

**Last Updated:** January 21, 2026
