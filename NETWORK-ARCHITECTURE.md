# Nick's Mining Pool - Network Architecture

## Overview

This document describes the complete network architecture for Nick's Mining Pool, including all IP addresses, port forwarding rules, and traffic flow.

---

## Network Topology

```
                    Internet (Global)
                           │
                           ▼
        ┌──────────────────────────────────┐
        │    Cloudflare DNS                │
        │    (DNS Resolution)              │
        └──────────────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────┐
        │    108.249.26.52                 │
        │    (External/Public IP)          │
        │    Your ISP Router               │
        └──────────────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────┐
        │    Router/Firewall               │
        │    NAT + Port Forwarding         │
        └──────────────────────────────────┘
                           │
                  ┌────────┴────────┐
                  │                 │
                  ▼                 ▼
    ┌─────────────────────┐   ┌─────────────────────┐
    │  NPM Server         │   │  Mining Pool        │
    │  172.16.0.21        │──▶│  Backend            │
    │  (Proxy/Gateway)    │   │  172.16.111.20      │
    └─────────────────────┘   └─────────────────────┘
          Internal LAN              Internal LAN
```

---

## IP Address Schema

### External/Public

| IP Address | Type | Purpose |
|------------|------|---------|
| **108.249.26.52** | Public IPv4 | External-facing IP address (assigned by ISP) |
|  | | Registered in Cloudflare DNS |
|  | | All incoming connections arrive here first |

### Internal/Private (LAN)

| IP Address | Hostname | Purpose | Services |
|------------|----------|---------|----------|
| **172.16.0.21** | NPM Server | Nginx Proxy Manager | HTTP/HTTPS proxy, TCP streams |
| **172.16.111.20** | Pool Backend | Mining Pool Software | Stratum daemons (BTC, BCH, BSV, DOGE, XEC), Web UI |

---

## Port Mapping

### External → NPM Server (Port Forwarding Rules)

Configure these rules on your router/firewall to forward from 108.249.26.52 → 172.16.0.21:

| External Port | Protocol | Internal IP | Internal Port | Service |
|---------------|----------|-------------|---------------|---------|
| **80** | TCP | 172.16.0.21 | 80 | HTTP (for Let's Encrypt, redirects to HTTPS) |
| **443** | TCP | 172.16.0.21 | 443 | HTTPS (pool web interface) |
| **6002** | TCP | 172.16.0.21 | 6002 | Bitcoin Cash (BCH) stratum |
| **6003** | TCP | 172.16.0.21 | 6003 | Dogecoin (DOGE) stratum |
| **6004** | TCP | 172.16.0.21 | 6004 | Bitcoin (BTC) stratum |
| **6005** | TCP | 172.16.0.21 | 6005 | Bitcoin SV (BSV) stratum |
| **6007** | TCP | 172.16.0.21 | 6007 | eCash (XEC) stratum |

**Note:** Port 81 (NPM admin interface) should **NOT** be exposed externally. Access via internal LAN only or through VPN.

---

### NPM Server → Mining Pool Backend (Internal Forwarding)

NPM forwards traffic from external ports to internal mining pool backend:

| NPM Listen Port | Forward To | Purpose |
|-----------------|------------|---------|
| 80/443 | 172.16.111.20:8559 | Web interface (HTTP/HTTPS proxy) |
| 6002 | 172.16.111.20:6002 | BCH stratum (TCP stream) |
| 6003 | 172.16.111.20:6003 | DOGE stratum (TCP stream) |
| 6004 | 172.16.111.20:6004 | BTC stratum (TCP stream) |
| 6005 | 172.16.111.20:6005 | BSV stratum (TCP stream) |
| 6007 | 172.16.111.20:6007 | XEC stratum (TCP stream) |

---

## Traffic Flow Diagrams

### Web Interface Access (HTTP/HTTPS)

```
User Browser
    │
    │ DNS Lookup: nicksminingpool.com
    ▼
Cloudflare DNS
    │ Returns: Cloudflare Proxy IP (🟠 Orange Cloud)
    ▼
Cloudflare Edge Server
    │ DDoS protection, SSL, caching
    │ Forwards to: 108.249.26.52:443
    ▼
Router/Firewall (108.249.26.52)
    │ Port forwarding: 443 → 172.16.0.21:443
    ▼
NPM Server (172.16.0.21:443)
    │ SSL Termination (Let's Encrypt cert)
    │ Reverse Proxy to: 172.16.111.20:8559
    ▼
Mining Pool Backend (172.16.111.20:8559)
    │ Web UI serves page
    ▼
User sees pool website
```

**Key Points:**
- Web traffic benefits from Cloudflare protection (orange cloud)
- NPM handles SSL termination
- Backend serves plain HTTP on port 8559 (NPM re-encrypts)

---

### Mining Connection (Stratum TCP)

```
Miner Software
    │
    │ DNS Lookup: btc.nicksminingpool.com
    ▼
Cloudflare DNS
    │ Returns: 108.249.26.52 (⚫ Gray Cloud - DNS-only)
    ▼
Router/Firewall (108.249.26.52)
    │ Port forwarding: 6004 → 172.16.0.21:6004
    ▼
NPM Server (172.16.0.21:6004)
    │ TCP Stream forward to: 172.16.111.20:6004
    ▼
Mining Pool Backend (172.16.111.20:6004)
    │ Stratum daemon responds
    ▼
Miner receives work
```

**Key Points:**
- Mining traffic bypasses Cloudflare proxy (gray cloud DNS-only)
- Direct TCP connection with minimal latency
- NPM acts as TCP proxy/load balancer
- Backend handles actual stratum protocol

---

## DNS Configuration (Cloudflare)

### Recommended Setup

```
Cloudflare DNS Records:

Type  Name                          Content          Proxy Status  TTL
──────────────────────────────────────────────────────────────────────
A     nicksminingpool.com           108.249.26.52    🟠 Proxied    Auto
A     www.nicksminingpool.com       108.249.26.52    🟠 Proxied    Auto
A     btc.nicksminingpool.com       108.249.26.52    ⚫ DNS-only   Auto
A     bch.nicksminingpool.com       108.249.26.52    ⚫ DNS-only   Auto
A     bsv.nicksminingpool.com       108.249.26.52    ⚫ DNS-only   Auto
A     doge.nicksminingpool.com      108.249.26.52    ⚫ DNS-only   Auto
A     ecash.nicksminingpool.com     108.249.26.52    ⚫ DNS-only   Auto
```

**Alternative (Wildcard):**
```
Type  Name                          Content          Proxy Status  TTL
──────────────────────────────────────────────────────────────────────
A     *.nicksminingpool.com         108.249.26.52    ⚫ DNS-only   Auto
A     nicksminingpool.com           108.249.26.52    🟠 Proxied    Auto
A     www.nicksminingpool.com       108.249.26.52    🟠 Proxied    Auto
```

---

## Firewall Rules

### Router/Firewall (External)

**Incoming Rules (WAN → LAN):**
```
Allow TCP 80    from any to 172.16.0.21
Allow TCP 443   from any to 172.16.0.21
Allow TCP 6002  from any to 172.16.0.21
Allow TCP 6003  from any to 172.16.0.21
Allow TCP 6004  from any to 172.16.0.21
Allow TCP 6005  from any to 172.16.0.21
Allow TCP 6007  from any to 172.16.0.21

Block TCP 81    from any (NPM admin - internal access only)
```

**Outgoing Rules (LAN → WAN):**
```
Allow all established/related connections
```

---

### NPM Server Firewall (UFW)

```bash
# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow stratum ports
sudo ufw allow 6002/tcp  # BCH
sudo ufw allow 6003/tcp  # DOGE
sudo ufw allow 6004/tcp  # BTC
sudo ufw allow 6005/tcp  # BSV
sudo ufw allow 6007/tcp  # XEC

# Allow NPM admin ONLY from specific IP (optional but recommended)
sudo ufw allow from 172.16.0.0/24 to any port 81 proto tcp

# SSH access (if needed)
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable
```

---

### Mining Pool Backend Firewall

```bash
# Allow connections from NPM server only
sudo ufw allow from 172.16.0.21 to any port 6002 proto tcp
sudo ufw allow from 172.16.0.21 to any port 6003 proto tcp
sudo ufw allow from 172.16.0.21 to any port 6004 proto tcp
sudo ufw allow from 172.16.0.21 to any port 6005 proto tcp
sudo ufw allow from 172.16.0.21 to any port 6007 proto tcp
sudo ufw allow from 172.16.0.21 to any port 8559 proto tcp

# Allow outgoing connections (for blockchain sync)
sudo ufw default allow outgoing

# SSH access
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable
```

---

## Security Considerations

### Network Segmentation

**Isolation Strategy:**
1. **NPM Server (172.16.0.21)** - DMZ-like zone, exposed to internet
2. **Mining Pool Backend (172.16.111.20)** - Protected zone, only accessible via NPM

**Why this matters:**
- If NPM is compromised, attacker still needs to breach internal firewall
- Mining pool backend is not directly exposed to internet
- Can implement additional IDS/IPS between zones

---

### DDoS Protection Layers

```
Layer 1: Cloudflare (Web Traffic Only)
├─ Rate limiting
├─ Challenge pages
├─ IP reputation filtering
└─ Volumetric attack mitigation

Layer 2: Router/Firewall
├─ Connection rate limiting
├─ SYN flood protection
└─ Stateful packet inspection

Layer 3: NPM (Nginx)
├─ Connection limits per IP
├─ Timeout configuration
└─ Upstream health checks

Layer 4: Mining Pool Backend
├─ Application-level banning
├─ Invalid share detection
└─ Worker rate limiting
```

---

## High Availability Considerations

### Future Scaling Options

**Option 1: Multiple NPM Servers (Load Balancing)**
```
                 Internet
                     │
              ┌──────┴──────┐
              ▼             ▼
       NPM Server 1    NPM Server 2
       172.16.0.21     172.16.0.22
              │             │
              └──────┬──────┘
                     ▼
           Mining Pool Backend
            172.16.111.20
```

**Option 2: Multiple Pool Backends (Redundancy)**
```
           Internet
               │
               ▼
          NPM Server
          172.16.0.21
               │
        ┌──────┴──────┐
        ▼             ▼
   Pool Backend 1  Pool Backend 2
   172.16.111.20   172.16.111.21
```

**Option 3: Geographic Distribution**
```
    Internet
        │
        ▼
   GeoDNS Routing
        │
   ┌────┴────┐
   ▼         ▼
US East    US West
Server     Server
```

---

## Monitoring and Health Checks

### Critical Monitoring Points

1. **Router/Firewall**
   - Monitor: WAN connection status, bandwidth usage
   - Alert: WAN link down, bandwidth >80%

2. **NPM Server (172.16.0.21)**
   - Monitor: CPU, RAM, disk, network connections
   - Alert: CPU >80%, RAM >90%, disk >85%, connection count >5000

3. **Mining Pool Backend (172.16.111.20)**
   - Monitor: Stratum daemon status, blockchain sync, database
   - Alert: Daemon down, sync stopped, DB connection failed

4. **End-to-End**
   - Monitor: External connectivity tests (web + stratum)
   - Alert: External web unreachable, stratum port timeout

---

## Backup and Disaster Recovery

### Backup Strategy

**NPM Server:**
- Configuration: `/home/user/nginx-proxy-manager/data/`
- SSL Certificates: `/home/user/nginx-proxy-manager/letsencrypt/`
- Frequency: Daily
- Retention: 30 days

**Mining Pool Backend:**
- Pool configuration files
- Wallet keys (encrypted)
- Database dumps
- Frequency: Every 6 hours
- Retention: 90 days

### Disaster Recovery Plan

**Scenario 1: NPM Server Failure**
1. Restore NPM from backup on new server
2. Update internal IP if changed
3. Update router port forwarding rules
4. Verify SSL certificates
5. Test all services

**Scenario 2: Mining Pool Backend Failure**
1. Restore pool software and configuration
2. Restore database from latest dump
3. Sync blockchain (may take hours/days)
4. Verify wallet access
5. Resume pool operations

**Scenario 3: ISP IP Change (108.249.26.52 changes)**
1. Update Cloudflare DNS A records with new IP
2. Update router WAN IP (if static)
3. No internal changes needed
4. Propagation time: 5 minutes to 24 hours

---

## Network Performance Optimization

### Bandwidth Requirements

**Estimated Usage:**

| Service | Per Connection | With 100 Miners | With 1000 Miners |
|---------|----------------|-----------------|------------------|
| Web UI | 1-2 KB/s | 100-200 KB/s | 1-2 MB/s |
| Stratum (each coin) | 0.5-1 KB/s | 50-100 KB/s | 0.5-1 MB/s |
| Blockchain Sync | 1-10 MB/s | 1-10 MB/s | 1-10 MB/s |

**Total Bandwidth:**
- 100 miners: ~5 MB/s down, ~2 MB/s up
- 1000 miners: ~20 MB/s down, ~10 MB/s up

**Recommended Connection:**
- Minimum: 100 Mbps down / 50 Mbps up
- Recommended: 500 Mbps down / 100 Mbps up
- Enterprise: 1 Gbps+ symmetric

---

### Latency Optimization

**Tips:**
1. Place NPM and pool backend in same LAN (minimize hops)
2. Use wired Ethernet (not WiFi) for servers
3. Enable TCP BBR congestion control
4. Disable unnecessary services consuming bandwidth
5. Use QoS to prioritize mining traffic

---

## Troubleshooting Network Issues

### Test Connectivity

```bash
# From external network (test public access)
curl -I https://nicksminingpool.com
telnet btc.nicksminingpool.com 6004

# From NPM server (test internal routing)
nc -zv 172.16.111.20 6004
nc -zv 172.16.111.20 8559

# From mining pool backend (test outbound)
ping 8.8.8.8  # Test internet
ping 172.16.0.21  # Test NPM server
```

### Trace Network Path

```bash
# From external
traceroute nicksminingpool.com

# Check DNS resolution
dig nicksminingpool.com +short
dig btc.nicksminingpool.com +short

# Verify port forwarding
nmap -p 6004 108.249.26.52
```

---

## Diagram Legend

| Symbol | Meaning |
|--------|---------|
| 🟠 | Cloudflare Proxied (Orange Cloud) |
| ⚫ | DNS-only (Gray Cloud) |
| → | Network connection |
| │ | Traffic flow direction |
| ▼ | Downward traffic flow |

---

**Last Updated:** January 21, 2026
**Network Design Version:** 1.0
**Maintained By:** Nick's Mining Pool Operations Team
