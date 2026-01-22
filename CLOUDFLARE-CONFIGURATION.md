# Cloudflare Configuration Guide for Nick's Mining Pool

## Overview

This guide provides step-by-step instructions for configuring Cloudflare DNS for your mining pool with the correct proxy settings.

**CRITICAL:** Mining pools require **different proxy settings** for web vs stratum traffic!

---

## Why Cloudflare Configuration Matters

### The Problem
- Cloudflare's proxy (🟠 orange cloud) only works with HTTP/HTTPS
- Stratum mining uses raw TCP connections (not HTTP)
- Proxying stratum through Cloudflare = **BROKEN mining connections**

### The Solution
- **🟠 Orange Cloud (Proxied)**: Web interface only
- **⚫ Gray Cloud (DNS-only)**: All stratum mining subdomains

---

## Step-by-Step Configuration

### Step 1: Add Your Domain to Cloudflare

1. Log into [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Click **"Add a Site"**
3. Enter: `nicksminingpool.com`
4. Click **"Add site"**
5. Select **Free** plan
6. Click **"Continue"**

---

### Step 2: Configure Nameservers at Domain Registrar

Cloud flare will show you nameservers like:
```
sue.ns.cloudflare.com
tim.ns.cloudflare.com
```

**At your domain registrar** (Namecheap, GoDaddy, etc.):
1. Log into your registrar account
2. Find DNS settings for nicksminingpool.com
3. Change nameservers to the ones Cloudflare provided
4. Save changes

**Propagation time:** 5 minutes to 48 hours (usually <30 minutes)

---

### Step 3: Add DNS Records

In Cloudflare dashboard → **DNS** → **Records**

#### Main Website Records (🟠 Proxied)

**Record 1:**
- Type: `A`
- Name: `@` (or `nicksminingpool.com`)
- IPv4 address: `108.249.26.52`
- Proxy status: **🟠 Proxied** (click cloud icon until orange)
- TTL: `Auto`
- Click **Save**

**Record 2:**
- Type: `A`
- Name: `www`
- IPv4 address: `108.249.26.52`
- Proxy status: **🟠 Proxied**
- TTL: `Auto`
- Click **Save**

---

#### Stratum Mining Records (⚫ DNS-only)

**Record 3 (Bitcoin):**
- Type: `A`
- Name: `btc`
- IPv4 address: `108.249.26.52`
- Proxy status: **⚫ DNS only** (click cloud icon until gray)
- TTL: `Auto`
- Click **Save**

**Record 4 (Bitcoin Cash):**
- Type: `A`
- Name: `bch`
- IPv4 address: `108.249.26.52`
- Proxy status: **⚫ DNS only**
- TTL: `Auto`
- Click **Save**

**Record 5 (Bitcoin SV):**
- Type: `A`
- Name: `bsv`
- IPv4 address: `108.249.26.52`
- Proxy status: **⚫ DNS only**
- TTL: `Auto`
- Click **Save**

**Record 6 (Dogecoin):**
- Type: `A`
- Name: `doge`
- IPv4 address: `108.249.26.52`
- Proxy status: **⚫ DNS only**
- TTL: `Auto`
- Click **Save**

**Record 7 (eCash):**
- Type: `A`
- Name: `ecash`
- IPv4 address: `108.249.26.52`
- Proxy status: **⚫ DNS only**
- TTL: `Auto`
- Click **Save**

---

### Step 3 Alternative: Using Wildcard DNS

**Instead of individual records, use wildcard:**

**Wildcard Record:**
- Type: `A`
- Name: `*`
- IPv4 address: `108.249.26.52`
- Proxy status: **⚫ DNS only**
- TTL: `Auto`
- Click **Save**

**Then override for main domain:**
- (Keep the `@` and `www` records as 🟠 Proxied from Step 3)

**Advantage:** Don't need to add new DNS record for each new coin!

---

### Step 4: Configure SSL/TLS Settings

In Cloudflare → **SSL/TLS** → **Overview**

**SSL/TLS encryption mode:**
- Select: **Full (strict)** ✅

**Why:**
- Cloudflare encrypts connection to your server
- Validates your Let's Encrypt certificate
- Most secure option

**Other modes (DON'T use):**
- ❌ Off: No encryption at all
- ❌ Flexible: Cloudflare → Server is unencrypted
- ❌ Full: Doesn't validate your certificate

---

### Step 5: Enable Always Use HTTPS

In Cloudflare → **SSL/TLS** → **Edge Certificates**

Scroll to **Always Use HTTPS**
- Toggle: **On** ✅

**What this does:**
- Redirects `http://nicksminingpool.com` → `https://nicksminingpool.com`
- Ensures all web traffic is encrypted

---

### Step 6: Configure Security Settings

#### 6.1 Security Level

In Cloudflare → **Security** → **Settings**

**Security Level:**
- Recommended: **Medium** (balance between security and false positives)
- Can increase to **High** if under attack

---

#### 6.2 Challenge Passage (CAPTCHA)

**Challenge Passage:**
- Set to: **30 minutes**
- Users who pass challenge won't be challenged again for 30 minutes

---

#### 6.3 Browser Integrity Check

**Browser Integrity Check:**
- Toggle: **On** ✅
- Blocks known malicious browsers and bots

---

### Step 7: Disable Features That Break Mining Pools

#### 7.1 Rocket Loader (TURN OFF)

In Cloudflare → **Speed** → **Optimization**

**Rocket Loader:**
- Toggle: **Off** ❌

**Why:** Breaks JavaScript on pool interface

---

#### 7.2 Auto Minify

**Auto Minify:**
- JavaScript: **Off** ❌ (can break pool web UI)
- CSS: **On** ✅ (usually safe)
- HTML: **On** ✅ (usually safe)

---

### Step 8: Verify Configuration

#### Test DNS Resolution

```bash
# Main domain (should return Cloudflare IP)
dig nicksminingpool.com +short
# Example output: 104.21.x.x or 172.67.x.x (Cloudflare range)

# Stratum subdomains (should return YOUR IP)
dig btc.nicksminingpool.com +short
# Expected output: 108.249.26.52 (your external IP)
```

---

#### Test Web Interface

```bash
# Should return Cloudflare headers
curl -I https://nicksminingpool.com

# Look for:
# cf-ray: xxxx
# cf-cache-status: xxxx
# server: cloudflare
```

---

#### Test Stratum Connections

```bash
# Should connect directly (no Cloudflare)
telnet btc.nicksminingpool.com 6004
telnet bch.nicksminingpool.com 6002
telnet bsv.nicksminingpool.com 6005
telnet doge.nicksminingpool.com 6003
telnet ecash.nicksminingpool.com 6007
```

---

## Optional: Advanced Cloudflare Features

### Page Rules (Free Plan: 3 rules)

#### Rule 1: Force HTTPS

- **If the URL matches:** `http://nicksminingpool.com/*`
- **Then the settings are:** Always Use HTTPS
- **Order:** 1

---

#### Rule 2: Cache Pool Stats API

- **If the URL matches:** `nicksminingpool.com/api/stats*`
- **Then the settings are:**
  - Cache Level: Cache Everything
  - Edge Cache TTL: 5 minutes
- **Order:** 2

---

### Firewall Rules (Protect Admin Interface)

In Cloudflare → **Security** → **WAF** → **Create firewall rule**

**Rule Name:** Block Admin Access from Unknown IPs

**Expression:**
```
(http.request.uri.path contains "/admin" and ip.src ne YOUR_ADMIN_IP)
```

**Action:** Block

**Replace `YOUR_ADMIN_IP`** with your actual IP address

---

## Troubleshooting

### Issue 1: Miners Can't Connect

**Symptoms:**
- Web interface works fine
- Miners get "Connection refused"

**Check:**
```bash
dig btc.nicksminingpool.com +short
```

**Expected:** `108.249.26.52` (your IP)
**If you see:** `104.x.x.x` or `172.67.x.x` → Cloud is ORANGE (wrong!)

**Fix:**
1. Go to Cloudflare DNS
2. Find `btc.nicksminingpool.com` record
3. Click orange cloud until it turns GRAY
4. Wait 5 minutes for DNS propagation
5. Test again

---

### Issue 2: SSL Errors on Website

**Symptoms:**
- Certificate warnings in browser
- "Your connection is not private"

**Check:**
1. Cloudflare → SSL/TLS → Overview
2. Ensure mode is: **Full (strict)**
3. Check NPM has valid Let's Encrypt certificate
4. Try force-refreshing browser (Ctrl+Shift+R)

---

### Issue 3: Website Shows Error 525 or 526

**Error 525:** SSL handshake failed
**Error 526:** Invalid SSL certificate

**Fix:**
1. Ensure NPM has valid SSL certificate
2. In Cloudflare, temporarily set SSL to **Full** (not strict)
3. Once working, change back to **Full (strict)**

---

### Issue 4: High Latency on Mining Connections

**Symptoms:**
- Miners have high latency
- Stale shares increase

**Check:**
```bash
# Test direct connection
ping btc.nicksminingpool.com
```

**If latency is high:**
1. Verify DNS is gray cloud (not proxied)
2. Check your ISP connection
3. Verify router/firewall not causing delays

---

## Security Best Practices

### DO:
- ✅ Use 🟠 orange cloud for web interface
- ✅ Use ⚫ gray cloud for all stratum subdomains
- ✅ Set SSL/TLS to "Full (strict)"
- ✅ Enable "Always Use HTTPS"
- ✅ Enable "Browser Integrity Check"
- ✅ Create firewall rules to protect `/admin` paths

### DON'T:
- ❌ Proxy stratum traffic through Cloudflare (orange cloud)
- ❌ Use SSL mode "Flexible" (insecure)
- ❌ Enable "Rocket Loader" (breaks pool JavaScript)
- ❌ Enable "I'm Under Attack" mode globally (only for web if DDoS)
- ❌ Use "Development Mode" in production

---

## Maintenance

### Monitor Cloudflare Analytics

In Cloudflare → **Analytics & Logs** → **Traffic**

**Monitor:**
- Requests per hour
- Bandwidth usage
- Threats blocked
- SSL/TLS versions used

---

### Review Security Events

In Cloudflare → **Security** → **Events**

**Check for:**
- Blocked requests
- Challenge completions
- Country-based blocks

**Adjust firewall rules** if seeing too many false positives

---

## Quick Reference

### DNS Record Summary

| Record | Type | Name | Content | Proxy |
|--------|------|------|---------|-------|
| Web | A | @ | 108.249.26.52 | 🟠 Proxied |
| Web WWW | A | www | 108.249.26.52 | 🟠 Proxied |
| BTC Stratum | A | btc | 108.249.26.52 | ⚫ DNS-only |
| BCH Stratum | A | bch | 108.249.26.52 | ⚫ DNS-only |
| BSV Stratum | A | bsv | 108.249.26.52 | ⚫ DNS-only |
| DOGE Stratum | A | doge | 108.249.26.52 | ⚫ DNS-only |
| XEC Stratum | A | ecash | 108.249.26.52 | ⚫ DNS-only |

---

### SSL/TLS Settings

| Setting | Value |
|---------|-------|
| Encryption Mode | Full (strict) |
| Always Use HTTPS | On |
| Minimum TLS Version | 1.2 |
| Automatic HTTPS Rewrites | On |

---

### Speed Optimizations

| Setting | Value |
|---------|-------|
| Auto Minify JS | Off |
| Auto Minify CSS | On |
| Auto Minify HTML | On |
| Rocket Loader | Off |
| Brotli | On |

---

## Support Resources

- **Cloudflare Docs:** https://developers.cloudflare.com/
- **Cloudflare Community:** https://community.cloudflare.com/
- **Status Page:** https://www.cloudflarestatus.com/

---

**Last Updated:** January 21, 2026
**Guide Version:** 1.0
**For:** Nick's Mining Pool
