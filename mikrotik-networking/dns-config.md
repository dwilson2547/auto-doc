# DNS Configuration on MikroTik

MikroTik RouterOS includes a built-in DNS client and a caching DNS resolver. This guide covers configuring the DNS client, enabling the DNS cache for LAN clients, adding static DNS entries, and configuring DNS over HTTPS (DoH).

## Table of Contents

- [Overview](#overview)
- [DNS Client Configuration](#dns-client-configuration)
- [Enabling the DNS Cache (Local Resolver)](#enabling-the-dns-cache-local-resolver)
- [Static DNS Entries](#static-dns-entries)
  - [A Records](#a-records)
  - [CNAME Records](#cname-records)
  - [Wildcard Entries](#wildcard-entries)
- [DNS over HTTPS (DoH)](#dns-over-https-doh)
- [DNS Forwarding / Split-DNS](#dns-forwarding--split-dns)
- [Per-VLAN DNS with Firewall Rules](#per-vlan-dns-with-firewall-rules)
- [Blocking DNS Queries to External Resolvers](#blocking-dns-queries-to-external-resolvers)
- [DNS Cache Monitoring and Management](#dns-cache-monitoring-and-management)
- [WinBox / WebFig Configuration](#winbox--webfig-configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

RouterOS DNS configuration lives at `/ip dns`. The router can act as:

1. **DNS client** — resolves names for the router itself (hostname lookups in scripts, ping by name, etc.)
2. **DNS caching resolver** — forwards and caches DNS queries from LAN clients, reducing external lookups
3. **Authoritative source for static entries** — serves custom DNS records to LAN clients

---

## DNS Client Configuration

Set the upstream DNS servers the router uses for its own name resolution and to forward client queries:

```bash
/ip dns set \
    servers=1.1.1.1,8.8.8.8 \
    allow-remote-requests=no
```

| Parameter | Description |
|-----------|-------------|
| `servers` | Comma-separated list of upstream DNS servers |
| `allow-remote-requests` | If `yes`, the router answers DNS queries from other hosts |

View current DNS settings:

```bash
/ip dns print
```

Sample output:

```
             servers: 1.1.1.1,8.8.8.8
  dynamic-servers: 203.0.113.1        ← assigned by ISP via DHCP/PPPoE
   use-doh-server:
  verify-doh-cert: no
allow-remote-requests: yes
      max-udp-packet-size: 4096
      query-server-timeout: 2s
       query-total-timeout: 10s
      max-concurrent-queries: 100
max-concurrent-tcp-sessions: 20
         cache-size: 2048KiB
      cache-max-ttl: 1w
           cache-used: 48KiB
```

---

## Enabling the DNS Cache (Local Resolver)

To use the router as a DNS server for LAN clients, enable `allow-remote-requests`:

```bash
/ip dns set allow-remote-requests=yes
```

Then point LAN clients (or the DHCP server) to the router's LAN IP as their DNS server:

```bash
# Tell DHCP clients to use the router as their DNS server
/ip dhcp-server network set [find comment="LAN"] dns-server=192.168.1.1
```

> **Security note:** Ensure your firewall drops DNS queries on the WAN interface to prevent the router from acting as an open resolver. See [Blocking DNS Queries to External Resolvers](#blocking-dns-queries-to-external-resolvers).

---

## Static DNS Entries

Static entries allow the router to answer DNS queries for specific hostnames without forwarding to an upstream server.

### A Records

Map a hostname to an IP address:

```bash
/ip dns static add name=router.lan address=192.168.1.1 ttl=1d comment="Router management"
/ip dns static add name=nas.lan address=192.168.1.50 ttl=1h comment="NAS server"
/ip dns static add name=printer.lan address=192.168.1.60 ttl=1h comment="Office printer"
```

### CNAME Records

Create a canonical name alias:

```bash
/ip dns static add name=files.lan cname=nas.lan ttl=1h comment="Alias for NAS"
/ip dns static add name=backup.lan cname=nas.lan ttl=1h
```

### Wildcard Entries

Match any subdomain using a wildcard (`*`):

```bash
# All *.dev.lan resolves to 192.168.1.100
/ip dns static add name=*.dev.lan address=192.168.1.100 comment="Dev server wildcard"
```

View all static entries:

```bash
/ip dns static print
```

---

## DNS over HTTPS (DoH)

RouterOS v6.47+ supports DNS over HTTPS, which encrypts DNS traffic to the upstream resolver.

### Step 1: Obtain the DoH Server Certificate

Fetch the certificate chain from your preferred DoH provider:

```bash
/certificate import file-name=cacert.pem
```

Or use the built-in Let's Encrypt roots (available in newer RouterOS versions):

```bash
/tool fetch url=https://curl.se/ca/cacert.pem
/certificate import file-name=cacert.pem passphrase=""
```

### Step 2: Configure DoH

```bash
/ip dns set \
    use-doh-server=https://cloudflare-dns.com/dns-query \
    verify-doh-cert=yes
```

Popular DoH endpoints:

| Provider | DoH URL |
|----------|---------|
| Cloudflare | `https://cloudflare-dns.com/dns-query` |
| Google | `https://dns.google/dns-query` |
| Quad9 | `https://dns.quad9.net/dns-query` |
| NextDNS | `https://dns.nextdns.io/<profile-id>` |

> **Note:** When using DoH, clear the `servers` field or set it to empty so RouterOS only uses DoH: `/ip dns set servers=""`

---

## DNS Forwarding / Split-DNS

Send queries for specific domains to a different DNS server (e.g., an internal Active Directory DNS for `.corp.example.com`):

```bash
# Forward corp.example.com queries to internal AD DNS
/ip dns static add \
    name=corp.example.com \
    type=FWD \
    forward-to=10.0.0.53 \
    comment="AD internal DNS"
```

> The `FWD` type is available in RouterOS v7.1+. In v6, use a separate DNS server tool or set up policy routing to redirect specific queries.

---

## Per-VLAN DNS with Firewall Rules

Force clients on a specific VLAN to use the router's DNS resolver (even if they are configured to use another DNS server):

```bash
# Redirect all DNS queries from VLAN30 (clients) to the router
/ip firewall nat add \
    chain=dstnat \
    in-interface=vlan30 \
    protocol=udp \
    dst-port=53 \
    action=redirect \
    to-ports=53 \
    comment="Force DNS to router for VLAN30"

/ip firewall nat add \
    chain=dstnat \
    in-interface=vlan30 \
    protocol=tcp \
    dst-port=53 \
    action=redirect \
    to-ports=53 \
    comment="Force DNS (TCP) to router for VLAN30"
```

---

## Blocking DNS Queries to External Resolvers

Prevent LAN clients from bypassing the router's DNS resolver by querying external DNS servers directly:

```bash
# Drop outgoing DNS queries that are NOT destined for the router itself
/ip firewall filter add \
    chain=forward \
    protocol=udp \
    dst-port=53 \
    dst-address=!192.168.1.1 \
    action=drop \
    comment="Block external DNS bypass (UDP)"

/ip firewall filter add \
    chain=forward \
    protocol=tcp \
    dst-port=53 \
    dst-address=!192.168.1.1 \
    action=drop \
    comment="Block external DNS bypass (TCP)"
```

Also protect the router itself from being an open resolver on WAN:

```bash
/ip firewall filter add \
    chain=input \
    protocol=udp \
    dst-port=53 \
    in-interface=ether1 \
    action=drop \
    comment="Block DNS queries on WAN"

/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=53 \
    in-interface=ether1 \
    action=drop \
    comment="Block DNS (TCP) queries on WAN"
```

---

## DNS Cache Monitoring and Management

```bash
# View current cache contents
/ip dns cache print

# View all cache entries (including dynamic from DHCP leases)
/ip dns cache all print

# Flush the DNS cache
/ip dns cache flush

# Set maximum cache size (in KiB)
/ip dns set cache-size=4096

# Set maximum TTL for cached entries
/ip dns set cache-max-ttl=1d
```

---

## WinBox / WebFig Configuration

1. Open **WinBox** → **IP** → **DNS**
2. Set **Servers** (e.g., `1.1.1.1` and `8.8.8.8`)
3. Check **Allow Remote Requests** if the router should serve DNS to LAN clients
4. Click **Apply**

For static entries:
1. In the DNS dialog, click **Static** tab
2. Click **+** (Add)
3. Enter **Name** and **Address** (or **CNAME**)
4. Click **OK**

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Router cannot resolve names | No DNS servers configured | Set `/ip dns set servers=1.1.1.1,8.8.8.8` |
| LAN clients not resolving names | `allow-remote-requests=no` | Enable `/ip dns set allow-remote-requests=yes` |
| Static entry not returned | TTL cached old result | Flush cache: `/ip dns cache flush` |
| DoH not working | Certificate not trusted | Import CA certificate and set `verify-doh-cert=yes` |
| External DNS queries still working | Firewall rule order incorrect | Move drop rules above accept rules in `/ip firewall filter` |

```bash
# Test DNS resolution from the router
/resolve google.com server=1.1.1.1

# Check if router answers DNS queries
/tool dns-lookup google.com

# Capture DNS traffic for debugging
/tool sniffer quick interface=bridge1 ip-protocol=udp port=53
```
