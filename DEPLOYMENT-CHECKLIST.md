# Nick's Mining Pool - Deployment Checklist

## Pre-Deployment Checklist

### Infrastructure Requirements
- [ ] NPM Server (172.16.0.21) - Ubuntu/Debian installed
- [ ] Mining Pool Backend (172.16.111.20) - Pool software running
- [ ] External IP (108.249.26.52) - Static IP configured
- [ ] Router/firewall admin access
- [ ] Domain registered: nicksminingpool.com
- [ ] Cloudflare account created

### Access Requirements
- [ ] Root/sudo access to NPM server
- [ ] Root/sudo access to pool backend server
- [ ] SSH keys configured
- [ ] Router/firewall admin credentials
- [ ] Cloudflare account login

### Software Prerequisites
- [ ] Docker installed on NPM server
- [ ] Docker Compose installed on NPM server
- [ ] Mining pool software configured on backend
- [ ] Blockchain daemons synced on backend

---

## Phase 1: Network Configuration

### Router/Firewall Setup
- [ ] Port 80 forwarded: 108.249.26.52 → 172.16.0.21:80
- [ ] Port 443 forwarded: 108.249.26.52 → 172.16.0.21:443
- [ ] Port 6002 forwarded: 108.249.26.52 → 172.16.0.21:6002 (BCH)
- [ ] Port 6003 forwarded: 108.249.26.52 → 172.16.0.21:6003 (DOGE)
- [ ] Port 6004 forwarded: 108.249.26.52 → 172.16.0.21:6004 (BTC)
- [ ] Port 6005 forwarded: 108.249.26.52 → 172.16.0.21:6005 (BSV)
- [ ] Port 6007 forwarded: 108.249.26.52 → 172.16.0.21:6007 (XEC)
- [ ] Port 81 NOT forwarded (internal access only)
- [ ] Port forwarding rules tested and verified

---

## Phase 2: Cloudflare Configuration

### Domain Setup
- [ ] Domain added to Cloudflare
- [ ] Nameservers updated at registrar
- [ ] Nameserver propagation verified (dig command)

### DNS Records (🟠 = Proxied, ⚫ = DNS-only)
- [ ] A record: nicksminingpool.com → 108.249.26.52 [🟠]
- [ ] A record: www.nicksminingpool.com → 108.249.26.52 [🟠]
- [ ] A record: btc.nicksminingpool.com → 108.249.26.52 [⚫]
- [ ] A record: bch.nicksminingpool.com → 108.249.26.52 [⚫]
- [ ] A record: bsv.nicksminingpool.com → 108.249.26.52 [⚫]
- [ ] A record: doge.nicksminingpool.com → 108.249.26.52 [⚫]
- [ ] A record: ecash.nicksminingpool.com → 108.249.26.52 [⚫]
- [ ] DNS propagation verified (dig +short commands)

### Cloudflare SSL/TLS Settings
- [ ] SSL/TLS encryption mode: Full (strict)
- [ ] Always Use HTTPS: Enabled
- [ ] Minimum TLS Version: 1.2
- [ ] Automatic HTTPS Rewrites: Enabled

### Cloudflare Speed Settings
- [ ] Auto Minify JS: Disabled
- [ ] Auto Minify CSS: Enabled
- [ ] Auto Minify HTML: Enabled
- [ ] Rocket Loader: Disabled
- [ ] Brotli: Enabled

---

## Phase 3: NPM Server Setup

### Docker Installation
- [ ] Docker installed successfully
- [ ] Docker Compose installed successfully
- [ ] User added to docker group
- [ ] Logout/login completed (group changes active)
- [ ] Docker version verified

### NPM Deployment
- [ ] Directory created: ~/nginx-proxy-manager
- [ ] docker-compose.yml created with correct configuration
- [ ] NPM containers started successfully
- [ ] NPM containers running (docker-compose ps)
- [ ] NPM logs show no errors

### Firewall Configuration
- [ ] UFW installed
- [ ] Port 80 allowed
- [ ] Port 443 allowed
- [ ] Port 6002 allowed (BCH)
- [ ] Port 6003 allowed (DOGE)
- [ ] Port 6004 allowed (BTC)
- [ ] Port 6005 allowed (BSV)
- [ ] Port 6007 allowed (XEC)
- [ ] Port 81 restricted to internal network only
- [ ] UFW enabled
- [ ] Firewall rules verified

---

## Phase 4: NPM Configuration

### Admin Access
- [ ] NPM admin accessible at http://172.16.0.21:81
- [ ] Logged in with default credentials
- [ ] **Admin password changed** (CRITICAL!)
- [ ] Admin email updated

### TCP Streams Configuration
- [ ] Stream added: Port 6002 → 172.16.111.20:6002 (BCH)
- [ ] Stream added: Port 6003 → 172.16.111.20:6003 (DOGE)
- [ ] Stream added: Port 6004 → 172.16.111.20:6004 (BTC)
- [ ] Stream added: Port 6005 → 172.16.111.20:6005 (BSV)
- [ ] Stream added: Port 6007 → 172.16.111.20:6007 (XEC)
- [ ] All streams saved successfully

### Proxy Host Configuration
- [ ] Proxy host created for nicksminingpool.com
- [ ] Domain names: nicksminingpool.com, www.nicksminingpool.com
- [ ] Forward to: 172.16.111.20:8559
- [ ] SSL certificate requested successfully
- [ ] Let's Encrypt validation passed
- [ ] Force SSL enabled
- [ ] HTTP/2 support enabled

---

## Phase 5: Verification Testing

### DNS Resolution Tests
- [ ] `dig nicksminingpool.com +short` returns Cloudflare IP
- [ ] `dig btc.nicksminingpool.com +short` returns 108.249.26.52
- [ ] `dig bch.nicksminingpool.com +short` returns 108.249.26.52
- [ ] `dig bsv.nicksminingpool.com +short` returns 108.249.26.52
- [ ] `dig doge.nicksminingpool.com +short` returns 108.249.26.52
- [ ] `dig ecash.nicksminingpool.com +short` returns 108.249.26.52

### Web Interface Tests
- [ ] `curl -I http://nicksminingpool.com` redirects to HTTPS
- [ ] `curl -I https://nicksminingpool.com` returns 200 OK
- [ ] Browser: https://nicksminingpool.com shows pool website
- [ ] Browser: SSL certificate valid (green padlock)
- [ ] Browser: No mixed content warnings

### Stratum Connection Tests
- [ ] `telnet btc.nicksminingpool.com 6004` connects
- [ ] `telnet bch.nicksminingpool.com 6002` connects
- [ ] `telnet bsv.nicksminingpool.com 6005` connects
- [ ] `telnet doge.nicksminingpool.com 6003` connects
- [ ] `telnet ecash.nicksminingpool.com 6007` connects

### Stratum Protocol Tests
- [ ] Send test command to BTC port, receive JSON response
- [ ] Send test command to BCH port, receive JSON response
- [ ] Send test command to BSV port, receive JSON response
- [ ] Send test command to DOGE port, receive JSON response
- [ ] Send test command to XEC port, receive JSON response

### SSL Certificate Tests
- [ ] Certificate issuer verified (Let's Encrypt)
- [ ] Certificate expiry > 60 days
- [ ] Certificate covers all domains
- [ ] No certificate warnings in browser

---

## Phase 6: Post-Deployment

### Security Hardening
- [ ] NPM admin port (81) restricted to internal network/VPN
- [ ] SSH key-based authentication configured
- [ ] Password authentication disabled in SSH
- [ ] Fail2ban installed and configured
- [ ] Intrusion detection configured (optional)

### Monitoring Setup
- [ ] Log rotation configured
- [ ] NPM logs accessible and clean
- [ ] Resource monitoring configured (CPU, RAM, disk)
- [ ] Connection count monitoring configured
- [ ] Alert thresholds configured

### Backup Configuration
- [ ] Backup script created
- [ ] Backup scheduled (cron)
- [ ] Backup tested (restore verification)
- [ ] Backup retention policy configured (30 days)
- [ ] Offsite backup configured

### Documentation
- [ ] Configuration notes documented
- [ ] Admin credentials stored securely (password manager)
- [ ] Network diagram updated
- [ ] Runbook created for common issues
- [ ] Emergency contact list created

---

## Phase 7: Go-Live

### Final Checks
- [ ] All previous checklist items completed
- [ ] Test miner connection from external network
- [ ] Test web interface from external network
- [ ] All pool staff notified of go-live
- [ ] Announcement prepared for miners

### Go-Live Actions
- [ ] Announce pool launch
- [ ] Monitor connections for first 24 hours
- [ ] Check for errors in logs
- [ ] Verify miners receiving work
- [ ] Verify payouts processing correctly

---

## Post-Go-Live Monitoring (First Week)

### Daily Checks
- [ ] **Day 1:** All services running, no errors
- [ ] **Day 2:** Connection count stable
- [ ] **Day 3:** No firewall blocks, payouts working
- [ ] **Day 4:** SSL certificate valid, no warnings
- [ ] **Day 5:** Backup completed successfully
- [ ] **Day 6:** Resource usage within limits
- [ ] **Day 7:** No security incidents, all systems healthy

---

## Rollback Plan (If Deployment Fails)

### Emergency Rollback Checklist
- [ ] Stop NPM: `docker-compose down`
- [ ] Identify failure point (logs, monitoring)
- [ ] Restore from backup if needed
- [ ] Fix identified issue
- [ ] Test in staging environment
- [ ] Retry deployment
- [ ] Document lessons learned

---

## Sign-Off

### Deployment Team

| Role | Name | Signature | Date |
|------|------|-----------|------|
| **System Administrator** | __________ | __________ | ____/____/____ |
| **Network Administrator** | __________ | __________ | ____/____/____ |
| **Pool Operator** | __________ | __________ | ____/____/____ |

### Deployment Status

- **Deployment Start:** ____/____/____ __:__
- **Deployment Complete:** ____/____/____ __:__
- **Total Time:** _____ hours
- **Issues Encountered:** _____________________________
- **Status:** [ ] Success [ ] Failed [ ] Partial

---

## Quick Reference

### Key URLs
- **Pool Website:** https://nicksminingpool.com
- **NPM Admin:** http://172.16.0.21:81
- **Cloudflare:** https://dash.cloudflare.com

### Key IPs
- **External:** 108.249.26.52
- **NPM Server:** 172.16.0.21
- **Pool Backend:** 172.16.111.20

### Stratum Connections
- **BTC:** stratum+tcp://btc.nicksminingpool.com:6004
- **BCH:** stratum+tcp://bch.nicksminingpool.com:6002
- **BSV:** stratum+tcp://bsv.nicksminingpool.com:6005
- **DOGE:** stratum+tcp://doge.nicksminingpool.com:6003
- **XEC:** stratum+tcp://ecash.nicksminingpool.com:6007

---

**Checklist Version:** 1.0
**Last Updated:** January 21, 2026
**Status:** Ready for Deployment
