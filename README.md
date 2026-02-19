# 🌐 Citrix NetScaler ADC — Enterprise Field Guide

> Built from **3.6+ years** of hands-on TAC and Priority Support experience at **Citrix (CSG)** and **HCL Tech**, supporting enterprise customers across global GEOs on NetScaler VPX, MPX, SDX, BLX, and ADM.

This repository documents real-world configurations, troubleshooting methodologies, and RCA frameworks used in production environments — not lab theory.

---

## 📋 Table of Contents

1. [Platform Coverage](#1-platform-coverage)
2. [Load Balancing & Content Switching](#2-load-balancing--content-switching)
3. [NetScaler Gateway & VPN](#3-netscaler-gateway--vpn)
4. [SSL/TLS Configuration & Troubleshooting](#4-ssltls-configuration--troubleshooting)
5. [Web Application Firewall (WAF)](#5-web-application-firewall-waf)
6. [Authentication — nFactor, SAML, LDAP, RADIUS, Okta](#6-authentication--nfactor-saml-ldap-radius-okta)
7. [High Availability (HA)](#7-high-availability-ha)
8. [GSLB — Global Server Load Balancing](#8-gslb--global-server-load-balancing)
9. [Rewrite & Responder Policies](#9-rewrite--responder-policies)
10. [Citrix ADM & Cloud](#10-citrix-adm--cloud)
11. [Troubleshooting Methodology](#11-troubleshooting-methodology)
12. [Real-World RCA Scenarios](#12-real-world-rca-scenarios)
13. [Quick Command Reference](#13-quick-command-reference)

---

## 1. Platform Coverage

| Platform | Form Factor | Notes |
|---|---|---|
| **VPX** | Virtual appliance | Deployed on ESXi, Hyper-V, Azure, AWS, GCP |
| **MPX** | Physical hardware | High-throughput enterprise deployments |
| **SDX** | Multi-tenant hardware | Multiple VPX instances on single appliance |
| **BLX** | Software-defined | Runs on commodity Linux servers |
| **ADM / NetScaler Console** | Management plane | Centralized monitoring, StyleBooks, Analytics |
| **Citrix Cloud** | SaaS | Cloud-hosted ADM, integrated with NetScaler instances |

---

## 2. Load Balancing & Content Switching

### 2.1 Load Balancing Methods

| Method | Use Case |
|---|---|
| Round Robin | Default — even distribution |
| Least Connection | Best for long-lived sessions |
| Resource Based | CPU/memory-aware balancing |
| Source IP Hash | Session persistence without cookies |
| Least Response Time | Best for latency-sensitive apps |

### 2.2 Persistence Methods

```
# Cookie-based persistence (most common for web apps)
set lb vserver <VS_NAME> -persistenceType COOKIEINSERT -timeout 60

# Source IP persistence
set lb vserver <VS_NAME> -persistenceType SOURCEIP -timeout 30

# Clear persistence on a vserver
set lb vserver <VS_NAME> -persistenceType NONE
```

### 2.3 Health Monitors

```
# HTTP monitor — checks specific URL and response code
add lb monitor <MON_NAME> HTTP -respCode 200 -httpRequest "GET /health"
bind lb monitor <MON_NAME> <SERVICE_NAME>

# TCP monitor — basic port check
add lb monitor <MON_NAME> TCP

# Check monitor status
show lb monitor <MON_NAME>
show service <SERVICE_NAME>
```

### 2.4 Content Switching

```
# Create content switching vserver
add cs vserver <CSV_NAME> HTTP <VIP> 80

# Create policy — route /api/* to backend API servers
add cs policy <POL_NAME> -rule "HTTP.REQ.URL.STARTSWITH(\"/api\")"

# Bind policy to CS vserver
bind cs vserver <CSV_NAME> -policyName <POL_NAME> -targetLBVserver <LB_VS_NAME> -priority 100
```

### 💡 Common Issue — Service Showing DOWN Despite Backend Being Up
Check in this order:
1. `show service <SERVICE>` — check monitor state and last failure reason
2. `show lb monitor <MON_NAME>` — check what the monitor is actually receiving
3. Verify SNIP has connectivity to backend: `ping -S <SNIP_IP> <BACKEND_IP>`
4. Check if firewall is blocking monitor probes between SNIP and backend

---

## 3. NetScaler Gateway & VPN

### 3.1 ICA Proxy — Citrix Virtual Apps & Desktops

ICA Proxy mode allows secure access to XenApp/XenDesktop without a full VPN tunnel. NetScaler acts as a reverse proxy for ICA traffic.

**Key components:**
- **Session Policy** — defines client experience (ICA Proxy vs Full VPN)
- **Session Profile** — sets StoreFront URL, SSO, split tunnel settings
- **AAA vserver** — handles authentication before granting access

```
# Check active Gateway sessions
show vpn session

# Clear a specific user's session
kill vpn session <SESSION_ID>

# Check ICA connections
show ica connection
```

### 3.2 SSL VPN Full Tunnel

```
# Check VPN session details
show vpn sessionAction <PROFILE_NAME>

# Verify split tunnel configuration
show vpn sessionPolicy
```

### 3.3 Endpoint Analysis (EPA)

EPA scans the client device before allowing access — checks for antivirus, OS version, domain membership, etc.

```
# Check EPA scan results for a session
show vpn epaProfile

# Common EPA failure — check what scan failed
show vpn session | grep EPA
```

### 💡 Common Issue — Users Getting Kicked Off Gateway Sessions
Check in order:
1. Session timeout settings in Session Profile (client idle timeout, session timeout)
2. SSL renegotiation — some clients drop on renegotiation
3. Check ASA/firewall UDP timeout if IPSec is involved
4. `show vpn session` — look for "Reason for termination" field

---

## 4. SSL/TLS Configuration & Troubleshooting

### 4.1 SSL Profile Best Practices

```
# Create a secure SSL profile (TLS 1.2+ only)
add ssl profile <PROFILE_NAME> -ssl3 DISABLED -tls1 DISABLED -tls11 DISABLED -tls12 ENABLED -tls13 ENABLED

# Bind profile to vserver
set ssl vserver <VS_NAME> -sslProfile <PROFILE_NAME>
```

### 4.2 Certificate Management

```
# Check certificate expiry
show ssl certKey | grep -i expir

# Update a certificate
update ssl certKey <CERTKEY_NAME> -cert <NEW_CERT_FILE> -key <NEW_KEY_FILE>

# Link intermediate cert to server cert
link ssl certKey <SERVER_CERT> <INTERMEDIATE_CERT>
```

### 4.3 SSL Offloading Verification

When SSL offload is working correctly:
- Client → NetScaler VIP: **HTTPS (443)**
- NetScaler SNIP → Backend: **HTTP (80)**

```
# Check SSL offload config on vserver
show ssl vserver <VS_NAME>

# Verify backend service is HTTP not HTTPS
show service <SERVICE_NAME>
```

### 4.4 Common SSL Issues & Fixes

| Issue | Likely Cause | Fix |
|---|---|---|
| `SSL handshake failure` | Cipher mismatch | Check and align cipher suites on both sides |
| `Certificate unknown` | Incomplete chain | Link intermediate cert to server cert |
| `Protocol version alert` | TLS version mismatch | Check client TLS support vs NetScaler SSL profile |
| `ERR_CERT_COMMON_NAME_INVALID` | SNI mismatch | Verify SNI settings on SSL vserver |
| Constant renegotiation drops | Renegotiation enabled | Disable renegotiation in SSL profile |

---

## 5. Web Application Firewall (WAF)

### 5.1 WAF Protection Profiles

WAF protects against OWASP Top 10 vulnerabilities including SQL injection, XSS, CSRF, and buffer overflow.

```
# Check WAF policy bindings on a vserver
show appfw profile <PROFILE_NAME>

# Show WAF violations in real time
show appfw stats

# Check WAF logs
shell "tail -f /var/log/ns.log | grep APPFW"
```

### 5.2 WAF Learning Mode

Before enforcing WAF, always run in **learning mode** first to capture legitimate traffic patterns.

```
# Enable learning for SQL injection
set appfw profile <PROFILE_NAME> -SQLInjectionAction learn log

# Review learned rules
show appfw learningdata <PROFILE_NAME> SQLInjection
```

### 5.3 Creating WAF Relaxation Rules (Whitelist)

When WAF blocks legitimate traffic:

```
# Add SQL injection relaxation for specific URL
bind appfw profile <PROFILE_NAME> -SQLInjectionRelaxation "field_name" "^/api/search$"

# Add XSS relaxation
bind appfw profile <PROFILE_NAME> -crossSiteScriptingRelaxation "comment_field" "^/submit$"
```

### 💡 Real Scenario — WAF Blocking Legitimate POST Requests
- **Symptom:** Users getting 403 on form submission
- **Filter:** `http.response.code == 403` in Wireshark + check `/var/log/ns.log` for `APPFW`
- **Finding:** WAF flagging special characters in form field as SQL injection
- **Fix:** Added relaxation rule for that specific field/URL after confirming with app team it was safe to whitelist

---

## 6. Authentication — nFactor, SAML, LDAP, RADIUS, Okta

### 6.1 nFactor Authentication

nFactor allows multi-step, conditional authentication flows — e.g., LDAP first, then OTP if coming from outside.

```
# Check nFactor policy flow
show authentication vserver <AAA_VS_NAME>
show authentication Policy
show authentication loginSchema
```

### 6.2 LDAP Troubleshooting

```
# Test LDAP connectivity from NetScaler
shell "ldapsearch -x -H ldap://<LDAP_IP> -D 'CN=svc_account,DC=domain,DC=com' -W -b 'DC=domain,DC=com'"

# Check LDAP action config
show authentication ldapAction <ACTION_NAME>

# Most common LDAP issues:
# 1. Wrong base DN
# 2. Service account locked/expired
# 3. Port 389 blocked by firewall (use 636 for LDAPS)
# 4. Group extraction misconfigured — check nested group settings
```

### 6.3 SAML / Okta Integration

```
# Check SAML IdP config
show authentication samlAction <ACTION_NAME>

# Common SAML issues:
# 1. Clock skew > 5 minutes between NetScaler and IdP
# 2. Assertion consumer URL mismatch
# 3. Certificate mismatch between SP metadata and IdP config
# 4. NameID format mismatch
```

### 6.4 RADIUS

```
# Test RADIUS from NetScaler CLI
shell "radtest <username> <password> <RADIUS_IP> 0 <shared_secret>"

# Check RADIUS action
show authentication radiusAction <ACTION_NAME>
```

---

## 7. High Availability (HA)

### 7.1 HA Status Check

```
# Check HA state
show ha node

# Expected output on primary:
# Node State: Primary
# Master State: Master
# Sync State: SUCCESS
```

### 7.2 Common HA Issues

| Issue | Cause | Fix |
|---|---|---|
| HA sync failing | Heartbeat blocked | Check firewall — UDP 3003 must be open between nodes |
| Split-brain | Both nodes think they're primary | Check network connectivity between HA nodes |
| Config not syncing | Force sync | `force ha failover` then `sync ha files all` |
| HA failover not happening | Health check misconfigured | Verify interface monitor settings |

```
# Force HA sync
sync ha files all

# Force failover (run on primary)
force ha failover

# Check HA sync status
show ha files
```

---

## 8. GSLB — Global Server Load Balancing

GSLB distributes traffic across multiple data centers for global redundancy and disaster recovery.

```
# Check GSLB site status
show gslb site

# Check GSLB service health
show gslb service

# Check GSLB vserver
show gslb vserver <VS_NAME>

# Verify DNS resolution for GSLB domain
shell "nslookup <GSLB_DOMAIN> <ADNS_IP>"
```

### Common GSLB Issues
- **MEP (Metric Exchange Protocol) down** — check UDP 3011 open between sites
- **ADNS not responding** — verify ADNS service is UP on NetScaler
- **Wrong site selected** — check LB method (RTT, static proximity, round robin)

---

## 9. Rewrite & Responder Policies

### 9.1 Rewrite — Header Insertion

```
# Insert X-Forwarded-For header
add rewrite action <ACT_NAME> insert_http_header X-Forwarded-For "CLIENT.IP.SRC"
add rewrite policy <POL_NAME> TRUE <ACT_NAME>
bind lb vserver <VS_NAME> -policyName <POL_NAME> -type REQUEST -priority 100
```

### 9.2 Rewrite — URL Redirect

```
# Redirect HTTP to HTTPS
add responder action <ACT_NAME> redirect "\"https://\" + HTTP.REQ.HOSTNAME + HTTP.REQ.URL"
add responder policy <POL_NAME> "HTTP.REQ.IS_VALID" <ACT_NAME>
bind lb vserver <VS_NAME> -policyName <POL_NAME> -type REQUEST
```

### 9.3 Responder — Block Specific IPs

```
# Return 403 for a specific IP
add responder action <ACT_NAME> respondwith "\"HTTP/1.1 403 Forbidden\r\n\r\n\""
add responder policy <POL_NAME> "CLIENT.IP.SRC.EQ(\"10.0.0.5\")" <ACT_NAME>
```

---

## 10. Citrix ADM & Cloud

```
# Common ADM use cases:
# - StyleBooks for templated deployments
# - Analytics for per-app performance metrics
# - Config audit and change tracking
# - Certificate expiry monitoring across all instances
# - MAS (Mass configuration push)

# Check ADM agent connectivity from NetScaler
show mas

# Verify ADM instance registration
show cloudprofile
```

---

## 11. Troubleshooting Methodology

This is the framework used for every escalation and war room case.

### Step 1 — Reproduce & Scope
- Is it all users or specific users?
- Is it all backends or specific servers?
- When did it start? Any changes (maintenance, cert renewal, firmware update)?

### Step 2 — Capture & Correlate
- Wireshark/tcpdump on **client side**, **NetScaler SNIP side**, and **backend side** simultaneously
- Fiddler (HAR file) from client browser for HTTP-level issues
- NetScaler ns.log and newnslog for internal events

```
# Capture on NetScaler
start nstrace -size 0 -mode RX TX -filter "SOURCEIP == <CLIENT_IP>"
stop nstrace
```

### Step 3 — Isolate the Layer
- Is it DNS? → Check `dns` filter in Wireshark
- Is it TCP? → Check `tcp.analysis.flags`
- Is it SSL? → Check `tls.alert_message`
- Is it application? → Check HTTP response codes

### Step 4 — RCA Documentation
Every P1/P2 case gets a full RCA with:
- **Timeline** of events
- **Root cause** (not just symptoms)
- **Immediate fix** applied
- **Permanent fix** and preventive measures

---

## 12. Real-World RCA Scenarios

### Scenario 1 — SSL VPN Sessions Dropping Every Hour
- **Symptom:** Remote users disconnected exactly at 60-minute mark
- **Capture:** `tls.alert_message` + `tcp.flags.reset == 1`
- **Finding:** ASA session timeout set to 3600s — RST sent at exactly 1 hour
- **Fix:** Increased ASA VPN session timeout, enabled DPD keepalives on client profile

### Scenario 2 — Intermittent 502 Bad Gateway
- **Symptom:** Random users getting 502, refreshing fixed it
- **Capture:** `tcp.flags.reset == 1` from SNIP IP
- **Finding:** One of three backend servers sending RST — app process crashed on that node
- **Fix:** Disabled server in LB, restarted app service, re-enabled after health check passed. Added better application-level health monitor to detect this faster.

### Scenario 3 — SAML SSO Failing for External Users Only
- **Symptom:** Internal users could authenticate, external users got SAML error
- **Finding:** Clock skew of 7 minutes between NetScaler and Okta IdP (NTP misconfigured on NetScaler)
- **Fix:** Corrected NTP server config on NetScaler — clock synced, SAML assertions accepted

### Scenario 4 — HA Failover Not Working During Maintenance
- **Symptom:** Primary node failed, secondary didn't take over
- **Finding:** Firewall rule had been updated to block UDP 3003 between HA nodes — heartbeat failing silently
- **Fix:** Restored firewall rule, validated HA heartbeat with `show ha node`, scheduled controlled failover test

### Scenario 5 — WAF Causing Performance Degradation
- **Symptom:** Application response times increased 3x after WAF was enabled
- **Finding:** WAF learning mode was logging every single request to disk — disk I/O saturation
- **Fix:** Disabled learning mode after rules were finalized, switched to enforce-only mode

---

## 13. Quick Command Reference

```bash
# ── SYSTEM ─────────────────────────────────────────
show version                          # Firmware version
show ns ip                            # All IP addresses (NSIP, SNIP, VIP)
show interface                        # Interface status
show ha node                          # HA state

# ── LOAD BALANCING ─────────────────────────────────
show lb vserver <NAME>                # vserver status and config
show service <NAME>                   # service/backend status
show lb monitor <NAME>                # monitor details
stat lb vserver <NAME>                # live traffic stats

# ── SSL ────────────────────────────────────────────
show ssl vserver <NAME>               # SSL config on vserver
show ssl certKey                      # All certs and expiry
show ssl profile <NAME>               # SSL profile details

# ── GATEWAY / VPN ──────────────────────────────────
show vpn session                      # Active VPN sessions
show ica connection                   # Active ICA connections
kill vpn session <ID>                 # Terminate a session

# ── WAF ────────────────────────────────────────────
show appfw profile <NAME>             # WAF profile config
show appfw stats                      # WAF hit statistics

# ── LOGS & TRACING ─────────────────────────────────
shell tail -f /var/log/ns.log         # Live system log
start nstrace -size 0 -mode RX TX    # Start packet capture
stop nstrace                          # Stop capture
shell nstcpdump.sh -i any             # Quick tcpdump

# ── HA ─────────────────────────────────────────────
show ha node                          # HA status
sync ha files all                     # Force config sync
force ha failover                     # Trigger failover

# ── GSLB ───────────────────────────────────────────
show gslb vserver <NAME>              # GSLB vserver status
show gslb site                        # Site health and MEP status
```

---

## About

Built by **Kruthik K V** — Network Security Engineer with 3.6+ years of enterprise TAC experience at Citrix (CSG) and HCL Tech, supporting NetScaler deployments on VPX, MPX, SDX, BLX, ADM, and public cloud (Azure, AWS, GCP).

📧 kruthik.39t@gmail.com
🔗 [LinkedIn](https://www.linkedin.com/in/kruthik-k-v-3115ba21a)
🔗 [GitHub Profile](https://github.com/Kruthik-NetworkSecurity)
🦈 [Wireshark Playbook](https://github.com/Kruthik-NetworkSecurity/Wireshark-Playbook)


