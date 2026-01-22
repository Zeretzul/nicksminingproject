# Nick's Mining Pool - NPM Configuration

Complete Nginx Proxy Manager setup for multi-coin cryptocurrency mining pool.

## 🎯 Overview

This repository contains ready-to-deploy configurations for running a mining pool with:
- **5 Cryptocurrencies**: Bitcoin (BTC), Bitcoin Cash (BCH), Bitcoin SV (BSV), Dogecoin (DOGE), eCash (XEC)
- **Nginx Proxy Manager**: TCP stream forwarding for stratum mining connections
- **SSL/TLS Support**: Automatic Let's Encrypt certificates
- **Web Interface**: HTTPS proxy for pool website

---

## 🚀 Quick Start

**Total setup time: 10-15 minutes**

```bash
# 1. Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh

# 2. Clone/download this repository
cd ~/
git clone <your-repo-url> nicksminingpool
cd nicksminingpool

# 3. Deploy NPM
docker-compose up -d

# 4. Configure firewall
sudo ufw allow 80,81,443,6002,6003,6004,6005,6007/tcp

# 5. Access admin panel
# Open browser: http://YOUR_SERVER_IP:81
# Login: admin@example.com / changeme (CHANGE IMMEDIATELY!)

# 6. Configure streams and proxy host
# See QUICKSTART.md for detailed instructions
```

---

## 📋 What's Included

### Configuration Files

| File | Description |
|------|-------------|
| **docker-compose.yml** | Ready-to-use NPM Docker setup |
| **stratum-streams.conf** | Nginx stream configuration for mining ports |
| **QUICKSTART.md** | 5-minute setup guide ⭐ START HERE |
| **NPM-CONFIGURATION.md** | Complete setup documentation |
| **MULTI-POOL-DNS-CONFIGURATION.md** | DNS setup options and best practices |
| **STRATUM-NGINX-PROXY-MANAGER-GUIDE.md** | Comprehensive stratum proxy guide |
| **README.md** | This file |

---

## 🌐 Pool Configuration

### Mining Ports (TCP Streams)

| Coin | Domain | Port | Backend |
|------|--------|------|---------|
| **Bitcoin (BTC)** | btc.nicksminingpool.com | 6004 | 172.16.111.20:6004 |
| **Bitcoin Cash (BCH)** | bch.nicksminingpool.com | 6002 | 172.16.111.20:6002 |
| **Bitcoin SV (BSV)** | bsv.nicksminingpool.com | 6005 | 172.16.111.20:6005 |
| **Dogecoin (DOGE)** | doge.nicksminingpool.com | 6003 | 172.16.111.20:6003 |
| **eCash (XEC)** | ecash.nicksminingpool.com | 6007 | 172.16.111.20:6007 |

### Web Interface (HTTP/HTTPS Proxy)

| Domain | Port | Backend |
|--------|------|---------|
| **nicksminingpool.com** | 80/443 | 172.16.111.20:8559 |

---

## 🔧 Installation Methods

### Method 1: Using NPM GUI (Recommended)

**For NPM version 2.10.0+** with built-in Streams support:

1. Access NPM admin panel (port 81)
2. Navigate to **Streams** tab
3. Add stream for each coin (see QUICKSTART.md)
4. Navigate to **Proxy Hosts** tab
5. Add proxy host for web interface

**Time:** 5 minutes
**Difficulty:** Easy ⭐

---

### Method 2: Manual Configuration

**For older NPM versions** without Streams GUI:

1. Deploy NPM with docker-compose
2. Copy `stratum-streams.conf` to container
3. Include in nginx.conf
4. Configure proxy host via GUI

**Time:** 10 minutes
**Difficulty:** Medium

See **NPM-CONFIGURATION.md** for detailed instructions.

---

## 📡 DNS Configuration

### Required DNS Records

**Option A: Individual Records**
```
A     btc.nicksminingpool.com      → YOUR_NPM_SERVER_IP
A     bch.nicksminingpool.com      → YOUR_NPM_SERVER_IP
A     bsv.nicksminingpool.com      → YOUR_NPM_SERVER_IP
A     doge.nicksminingpool.com     → YOUR_NPM_SERVER_IP
A     ecash.nicksminingpool.com    → YOUR_NPM_SERVER_IP
A     nicksminingpool.com          → YOUR_NPM_SERVER_IP
A     www.nicksminingpool.com      → YOUR_NPM_SERVER_IP
```

**Option B: Wildcard (Recommended)**
```
A     *.nicksminingpool.com        → YOUR_NPM_SERVER_IP
A     nicksminingpool.com          → YOUR_NPM_SERVER_IP
```

See **MULTI-POOL-DNS-CONFIGURATION.md** for detailed DNS options.

---

## 🔒 Security Features

- ✅ **SSL/TLS Encryption**: Automatic Let's Encrypt certificates
- ✅ **HTTP to HTTPS Redirect**: Forced SSL on web interface
- ✅ **Firewall Configuration**: UFW rules for security
- ✅ **Connection Timeouts**: Prevent resource exhaustion
- ✅ **Health Checks**: Automatic backend failure detection
- ✅ **Rate Limiting**: Built-in connection management

---

## 🧪 Testing

### Test Stratum Connections

```bash
# Test each pool port
telnet btc.nicksminingpool.com 6004
telnet bch.nicksminingpool.com 6002
telnet bsv.nicksminingpool.com 6005
telnet doge.nicksminingpool.com 6003
telnet ecash.nicksminingpool.com 6007

# Test stratum protocol
echo '{"id": 1, "method": "mining.subscribe", "params": []}' | nc btc.nicksminingpool.com 6004
```

### Test Web Interface

```bash
# Test HTTP (should redirect to HTTPS)
curl -I http://nicksminingpool.com

# Test HTTPS (should return 200 OK)
curl -I https://nicksminingpool.com

# Browser test
# Visit: https://nicksminingpool.com
```

---

## 🛠️ Management

### Common Commands

```bash
# View logs
docker-compose logs -f

# Restart NPM
docker-compose restart

# Stop NPM
docker-compose down

# Start NPM
docker-compose up -d

# Update NPM
docker-compose pull
docker-compose up -d

# Check status
docker-compose ps
```

### Monitor Connections

```bash
# Active mining connections
watch 'netstat -an | grep -E "(6002|6003|6004|6005|6007)" | grep ESTABLISHED | wc -l'

# Per-port breakdown
netstat -an | grep ESTABLISHED | grep -E "(6002|6003|6004|6005|6007)" | awk '{print $4}' | sort | uniq -c
```

---

## 📖 Documentation

### Getting Started
- **QUICKSTART.md** - Start here for fast setup
- **NPM-CONFIGURATION.md** - Complete configuration guide
- **MULTI-POOL-DNS-CONFIGURATION.md** - DNS setup options

### Advanced Topics
- **STRATUM-NGINX-PROXY-MANAGER-GUIDE.md** - Comprehensive guide
  - Installing mining pool software (NOMP)
  - SSL/TLS configuration
  - Load balancing
  - DDoS protection
  - Monitoring and troubleshooting

---

## 🚨 Troubleshooting

### NPM Won't Start

```bash
# Check Docker is running
sudo systemctl status docker

# Check for port conflicts
sudo netstat -tlnp | grep -E '(80|81|443)'

# View detailed logs
docker logs nginx-proxy-manager --tail 100
```

### Stratum Port Not Responding

```bash
# Test backend directly
nc -zv 172.16.111.20 6004

# Check firewall
sudo ufw status | grep 6004

# Check if NPM is listening
docker exec nginx-proxy-manager netstat -tlnp | grep 6004
```

### SSL Certificate Issues

```bash
# Verify DNS propagation
dig nicksminingpool.com +short

# Ensure port 80 is accessible
sudo ufw allow 80/tcp

# Check Let's Encrypt logs
docker logs nginx-proxy-manager | grep -i certbot
```

**See NPM-CONFIGURATION.md for complete troubleshooting guide.**

---

## 📊 Architecture

```
        Internet (Miners & Users)
                  │
                  ▼
      ┌────────────────────────┐
      │  108.249.26.52         │
      │  (External IP)         │
      │  Cloudflare DNS        │
      └────────────┬───────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  Router/Firewall       │
      │  Port Forwarding       │
      └────────────┬───────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│  Nginx Proxy Manager                     │
│  172.16.0.21 (Internal LAN)             │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │  TCP Streams (Mining Ports)        │ │
│  │  • 6002 → 172.16.111.20:6002 (BCH) │ │
│  │  • 6003 → 172.16.111.20:6003 (DOGE)│ │
│  │  • 6004 → 172.16.111.20:6004 (BTC) │ │
│  │  • 6005 → 172.16.111.20:6005 (BSV) │ │
│  │  • 6007 → 172.16.111.20:6007 (XEC) │ │
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │  HTTP Proxy (Web Interface)        │ │
│  │  • 80/443 → 172.16.111.20:8559    │ │
│  └────────────────────────────────────┘ │
└──────────────────┬───────────────────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  Mining Pool Backend   │
      │  172.16.111.20         │
      │  (Internal LAN)        │
      │                        │
      │  Stratum Servers:      │
      │  • BTC  :6004         │
      │  • BCH  :6002         │
      │  • BSV  :6005         │
      │  • DOGE :6003         │
      │  • XEC  :6007         │
      │                        │
      │  Web Interface :8559   │
      └────────────────────────┘
```

---

## 💻 Miner Connection Examples

### Bitcoin (BTC)
```bash
cgminer -o stratum+tcp://btc.nicksminingpool.com:6004 \
  -u YOUR_WALLET_ADDRESS.worker1 \
  -p x
```

### Bitcoin Cash (BCH)
```bash
cgminer -o stratum+tcp://bch.nicksminingpool.com:6002 \
  -u YOUR_WALLET_ADDRESS.worker1 \
  -p x
```

### Bitcoin SV (BSV)
```bash
cgminer -o stratum+tcp://bsv.nicksminingpool.com:6005 \
  -u YOUR_WALLET_ADDRESS.worker1 \
  -p x
```

### Dogecoin (DOGE)
```bash
cgminer -o stratum+tcp://doge.nicksminingpool.com:6003 \
  -u YOUR_WALLET_ADDRESS.worker1 \
  -p x
```

### eCash (XEC)
```bash
cgminer -o stratum+tcp://ecash.nicksminingpool.com:6007 \
  -u YOUR_WALLET_ADDRESS.worker1 \
  -p x
```

---

## 📦 Requirements

### System Requirements
- Ubuntu 20.04/22.04 LTS or Debian 11/12
- 2+ CPU cores
- 4+ GB RAM
- 20+ GB storage
- Static IP or DDNS

### Software Requirements
- Docker 20.10+
- Docker Compose 2.0+
- Root/sudo access

### Network Requirements
- Ports 80, 443 (web interface)
- Ports 6002, 6003, 6004, 6005, 6007 (stratum)
- Port 81 (NPM admin - restrict to admin IPs)

---

## 🔄 Updates & Maintenance

### Update NPM

```bash
cd ~/nicksminingpool
docker-compose pull
docker-compose up -d
```

### Backup Configuration

```bash
# Backup NPM data
tar -czf npm-backup-$(date +%Y%m%d).tar.gz data/ letsencrypt/

# Restore from backup
tar -xzf npm-backup-YYYYMMDD.tar.gz
docker-compose up -d
```

### Certificate Renewal

NPM automatically renews Let's Encrypt certificates. No manual action required.

**Check renewal status:**
```bash
docker exec nginx-proxy-manager certbot certificates
```

---

## 📝 License

This configuration is provided as-is for educational and operational purposes.

**Software Credits:**
- [Nginx Proxy Manager](https://nginxproxymanager.com) - JPressHouse/Jamie Curnow
- [Let's Encrypt](https://letsencrypt.org) - Free SSL certificates
- [Docker](https://docker.com) - Container platform

---

## 🤝 Support

### Documentation
- Read **QUICKSTART.md** for quick setup
- Read **NPM-CONFIGURATION.md** for detailed configuration
- Read **STRATUM-NGINX-PROXY-MANAGER-GUIDE.md** for comprehensive guide

### Troubleshooting
1. Check NPM logs: `docker logs nginx-proxy-manager`
2. Test backend connectivity: `nc -zv 172.16.111.20 6004`
3. Verify DNS records: `dig btc.nicksminingpool.com +short`
4. Check firewall: `sudo ufw status`

### Useful Resources
- [NPM Documentation](https://nginxproxymanager.com/guide/)
- [Nginx Stream Module](https://nginx.org/en/docs/stream/ngx_stream_core_module.html)
- [Stratum Protocol](https://en.bitcoin.it/wiki/Stratum_mining_protocol)
- [Let's Encrypt Docs](https://letsencrypt.org/docs/)

---

## ✅ Checklist

Before going live, verify:

- [ ] Docker and Docker Compose installed
- [ ] NPM containers running (`docker-compose ps`)
- [ ] Firewall configured (ports 80, 81, 443, 6002-6007)
- [ ] DNS records pointing to your server
- [ ] NPM admin password changed
- [ ] Streams configured (all 5 coins)
- [ ] Proxy host configured (web interface)
- [ ] SSL certificate obtained (nicksminingpool.com)
- [ ] Stratum ports tested (telnet to each port)
- [ ] Web interface accessible (https://nicksminingpool.com)
- [ ] Miner connections tested
- [ ] Backups configured

---

## 📅 Changelog

### v1.0.0 - January 21, 2026
- Initial release
- Support for 5 cryptocurrencies (BTC, BCH, BSV, DOGE, XEC)
- Nginx Proxy Manager configuration
- Automatic SSL/TLS with Let's Encrypt
- Complete documentation

---

**Last Updated:** January 21, 2026
**Version:** 1.0.0
**Maintainer:** Nick's Mining Pool

---

🚀 **Ready to deploy?** Start with **QUICKSTART.md**!
