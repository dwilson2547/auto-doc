# RIP and Dynamic Routing Protocols on MikroTik

MikroTik RouterOS supports several dynamic routing protocols, allowing routers to automatically discover and maintain routes to network destinations. This guide covers **RIP** (Routing Information Protocol) in depth and provides an introduction to **OSPF** and **BGP**, the most common dynamic routing protocols used with MikroTik equipment.

## Table of Contents

- [Overview](#overview)
- [Choosing a Routing Protocol](#choosing-a-routing-protocol)
- [RIP (Routing Information Protocol)](#rip-routing-information-protocol)
  - [RIP Concepts](#rip-concepts)
  - [Enabling RIP (RIPv2)](#enabling-rip-ripv2)
  - [Adding Networks to RIP](#adding-networks-to-rip)
  - [RIP Neighbors](#rip-neighbors)
  - [RIP Authentication](#rip-authentication)
  - [Redistributing Static and Connected Routes](#redistributing-static-and-connected-routes)
  - [RIP Timers](#rip-timers)
  - [Verifying RIP](#verifying-rip)
- [OSPF (Open Shortest Path First)](#ospf-open-shortest-path-first)
  - [OSPF Concepts](#ospf-concepts)
  - [Basic OSPF Configuration (RouterOS v7)](#basic-ospf-configuration-routeros-v7)
  - [OSPF Areas](#ospf-areas)
  - [OSPF Authentication](#ospf-authentication)
  - [Route Redistribution into OSPF](#route-redistribution-into-ospf)
  - [Verifying OSPF](#verifying-ospf)
- [BGP (Border Gateway Protocol)](#bgp-border-gateway-protocol)
  - [BGP Concepts](#bgp-concepts)
  - [Basic eBGP Configuration](#basic-ebgp-configuration)
  - [BGP Filtering with Route Filters](#bgp-filtering-with-route-filters)
  - [Verifying BGP](#verifying-bgp)
- [Comparing Protocols: Quick Reference](#comparing-protocols-quick-reference)
- [Troubleshooting](#troubleshooting)

---

## Overview

Dynamic routing protocols automate the exchange of routing information between routers, eliminating manual route management in larger networks. RouterOS implements:

| Protocol | Version | Standard | RouterOS Path |
|----------|---------|----------|--------------|
| RIP | v1, v2, RIPng (IPv6) | RFC 2453 | `/routing rip` |
| OSPF | v2 (IPv4), v3 (IPv6) | RFC 2328 / RFC 5340 | `/routing ospf` |
| BGP | v4 | RFC 4271 | `/routing bgp` |

> **RouterOS v7 note:** The routing subsystem was significantly redesigned in v7. This guide provides v7 syntax (under `/routing`) alongside v6 equivalents where they differ.

---

## Choosing a Routing Protocol

| Protocol | Best For | Not Suitable For |
|----------|---------|-----------------|
| **Static** | Simple, small networks; default gateway | Large or frequently changing topologies |
| **RIP** | Small–medium networks (< 15 hops); simple config | Large networks; slow convergence |
| **OSPF** | Enterprise networks; fast convergence; scalable | Very small networks (overhead not worth it) |
| **BGP** | Inter-AS routing; ISP peering; large-scale routing | Simple LAN routing |

---

## RIP (Routing Information Protocol)

### RIP Concepts

RIP is a distance-vector protocol that uses **hop count** as its metric.

| Property | Value |
|----------|-------|
| Metric | Hop count (1 = directly connected) |
| Maximum hops | 15 (16 = unreachable / infinity) |
| Update interval | 30 seconds |
| Hold-down timer | 180 seconds |
| Flush timer | 240 seconds |
| Convergence | Slow (minutes in worst case) |
| Authentication | MD5 or plain text (RIPv2 only) |

### Enabling RIP (RIPv2)

**RouterOS v7:**

```bash
# Create a RIP instance
/routing rip instance add \
    name=rip1 \
    version=2 \
    redistribute="" \
    comment="Main RIP instance"
```

**RouterOS v6:**

```bash
/routing rip set \
    distribute-default=never \
    metric-default=1 \
    redistribute-connected=no \
    redistribute-static=no \
    routing-table=main
```

### Adding Networks to RIP

Define which interfaces/networks RIP will advertise and listen on:

**RouterOS v7:**

```bash
# Add networks to advertise via RIP
/routing rip interface-template add \
    instance=rip1 \
    interfaces=ether2 \
    send=v2 \
    receive=v2 \
    comment="RIP on LAN ether2"

/routing rip interface-template add \
    instance=rip1 \
    interfaces=ether3 \
    send=v2 \
    receive=v2 \
    comment="RIP on ether3"
```

**RouterOS v6:**

```bash
/routing rip network add network=192.168.1.0/24
/routing rip network add network=10.0.0.0/8
```

### RIP Neighbors

For unicast RIP (when multicast is blocked, e.g., over WAN links):

**RouterOS v7:**

```bash
/routing rip neighbor add \
    instance=rip1 \
    address=10.0.0.2 \
    comment="Unicast RIP neighbor"
```

**RouterOS v6:**

```bash
/routing rip neighbor add neighbor=10.0.0.2
```

### RIP Authentication

Secure RIP updates with MD5 authentication to prevent rogue routers from injecting routes:

**RouterOS v7:**

```bash
/routing rip interface-template set [find interfaces=ether2] \
    auth=md5 \
    auth-key=MySecretKey123
```

**RouterOS v6:**

```bash
/routing rip interface set ether2 \
    authentication=md5 \
    authentication-key=MySecretKey123
```

### Redistributing Static and Connected Routes

Inject static routes or directly connected networks into RIP:

**RouterOS v7:**

```bash
/routing rip instance set rip1 redistribute=connected,static
```

**RouterOS v6:**

```bash
/routing rip set \
    redistribute-connected=yes \
    redistribute-static=yes \
    metric-default=1
```

### RIP Timers

Adjust update and hold-down timers (use with caution — must match all RIP peers):

**RouterOS v7:**

```bash
/routing rip instance set rip1 \
    update-interval=30 \
    timeout-interval=180 \
    garbage-collect-interval=120
```

### Verifying RIP

```bash
# View RIP routes learned from neighbors
/ip route print where routing-mark="" protocol=rip

# RouterOS v7: view RIP routes
/routing route print where protocol=rip

# View RIP neighbors and their status (v7)
/routing rip neighbor print

# View interfaces participating in RIP (v7)
/routing rip interface print

# View RIP statistics (v7)
/routing rip instance print detail
```

---

## OSPF (Open Shortest Path First)

### OSPF Concepts

OSPF is a link-state protocol that builds a complete topology map (LSDB) and uses Dijkstra's algorithm to calculate shortest paths.

| Property | Value |
|----------|-------|
| Metric | Cost (based on interface bandwidth) |
| Maximum hops | Unlimited |
| Convergence | Fast (seconds) |
| Authentication | Plain text or MD5 |
| Multicast | 224.0.0.5 (all OSPF routers), 224.0.0.6 (DR/BDR) |
| Hello interval | 10 seconds (default) |
| Dead interval | 40 seconds (default) |

### Basic OSPF Configuration (RouterOS v7)

```bash
# Step 1: Create OSPF instance
/routing ospf instance add \
    name=ospf1 \
    version=2 \
    router-id=1.1.1.1 \
    comment="Main OSPF instance"

# Step 2: Create backbone area (Area 0)
/routing ospf area add \
    name=backbone \
    area-id=0.0.0.0 \
    instance=ospf1

# Step 3: Add interfaces to OSPF area
/routing ospf interface-template add \
    instance=ospf1 \
    area=backbone \
    interfaces=ether2 \
    hello-interval=10s \
    dead-interval=40s \
    comment="OSPF on LAN"

/routing ospf interface-template add \
    instance=ospf1 \
    area=backbone \
    interfaces=ether3 \
    comment="OSPF on WAN link"
```

**RouterOS v6 equivalent:**

```bash
/routing ospf instance set default router-id=1.1.1.1
/routing ospf network add network=192.168.1.0/24 area=backbone
/routing ospf network add network=10.0.0.0/30 area=backbone
```

### OSPF Areas

Multi-area OSPF reduces LSDB size and improves scalability:

```bash
# Create area 1 (stub area)
/routing ospf area add \
    name=area1 \
    area-id=0.0.0.1 \
    instance=ospf1 \
    type=stub \
    comment="Stub area — no external LSAs"

# Add interfaces to area 1
/routing ospf interface-template add \
    instance=ospf1 \
    area=area1 \
    interfaces=ether4 \
    comment="OSPF area 1 interface"
```

| Area Type | External LSAs | Default Route |
|-----------|-------------|--------------|
| Backbone (area 0) | Yes | No |
| Standard | Yes | No |
| Stub | No | Injected by ABR |
| NSSA | External type 7 only | Optional |
| Totally Stub | No | Injected by ABR |

### OSPF Authentication

**RouterOS v7:**

```bash
/routing ospf interface-template set [find interfaces=ether2] \
    auth=md5 \
    auth-key=OSPFSecret
```

### Route Redistribution into OSPF

```bash
/routing ospf instance set ospf1 redistribute=connected,static
```

Specify a default metric for redistributed routes:

```bash
/routing ospf instance set ospf1 \
    redistribute=connected,static \
    metric-default=20 \
    metric-type=type2
```

### Verifying OSPF

```bash
# View OSPF neighbors and their state (v7)
/routing ospf neighbor print

# View OSPF routes
/routing route print where protocol=ospf

# View OSPF areas
/routing ospf area print

# View OSPF interface status
/routing ospf interface print

# View OSPF LSDB
/routing ospf lsa print
```

Expected neighbor states (from lowest to full adjacency):

```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```

`Full` means the adjacency is established and LSDBs are synchronized.

---

## BGP (Border Gateway Protocol)

### BGP Concepts

BGP is the routing protocol of the internet. It exchanges routing information between autonomous systems (AS).

| Property | Value |
|----------|-------|
| Metric | AS path length, MED, local preference, etc. |
| Type | eBGP (between AS), iBGP (within AS) |
| Port | TCP 179 |
| Convergence | Slow (policy-based) |
| Authentication | MD5 TCP session authentication |

### Basic eBGP Configuration

**Topology:** Your router (AS 65001) peers with ISP router (AS 65000) at `203.0.113.1`.

**RouterOS v7:**

```bash
# Create BGP connection to eBGP peer
/routing bgp connection add \
    name=isp-peer \
    remote.address=203.0.113.1 \
    remote.as=65000 \
    local.role=ebgp \
    as=65001 \
    hold-time=90 \
    keepalive-time=30 \
    comment="ISP eBGP peer"
```

**RouterOS v6:**

```bash
/routing bgp instance set default as=65001
/routing bgp peer add \
    name=isp-peer \
    remote-address=203.0.113.1 \
    remote-as=65000 \
    hold-time=90 \
    keepalive-time=30 \
    comment="ISP eBGP peer"
```

### Advertise Your Prefix to BGP

```bash
# v7: Create a network to advertise
/routing bgp network add network=198.51.100.0/24 comment="Our public prefix"

# v6:
/routing bgp network add network=198.51.100.0/24 synchronize=no
```

### BGP Filtering with Route Filters

Prevent accepting or sending routes you don't intend to:

**RouterOS v7:**

```bash
# Create a filter that only accepts routes from the ISP's address space
/routing filter rule add \
    chain=bgp-in \
    rule="if (dst in 0.0.0.0/0 && dst-len == 0) { accept }"

# Attach filter to BGP connection
/routing bgp connection set isp-peer input.filter=bgp-in
```

**RouterOS v6:**

```bash
# Accept only a default route from ISP
/routing filter add \
    chain=bgp-in \
    prefix=0.0.0.0/0 \
    prefix-length=0-0 \
    action=accept

/routing filter add chain=bgp-in action=discard

/routing bgp peer set isp-peer in-filter=bgp-in
```

### BGP Authentication

```bash
# v7
/routing bgp connection set isp-peer tcp-md5-key=BGPSecretKey

# v6
/routing bgp peer set isp-peer tcp-md5-key=BGPSecretKey
```

### Verifying BGP

```bash
# View BGP peers and their state (v7)
/routing bgp peer print

# View BGP sessions
/routing bgp connection print

# View BGP routes
/routing route print where protocol=bgp

# View BGP advertisements
/routing bgp advertisements print peer=isp-peer

# View received BGP routes
/routing bgp peer-info print
```

Expected BGP state progression:

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

`Established` means the session is up and routes are being exchanged.

---

## Comparing Protocols: Quick Reference

| Feature | RIP | OSPF | BGP |
|---------|-----|------|-----|
| Protocol type | Distance vector | Link-state | Path vector |
| Metric | Hop count | Cost (bandwidth) | AS path / attributes |
| Max hops | 15 | Unlimited | Unlimited |
| Convergence | Slow (30–240s) | Fast (< 10s) | Slow (policy-based) |
| Scalability | Small networks | Large enterprise | Internet-scale |
| Config complexity | Simple | Moderate | Complex |
| Authentication | MD5 (v2) | MD5 | TCP MD5 |
| VLAN support | Yes (with routing) | Yes | Yes |
| RouterOS path (v7) | `/routing rip` | `/routing ospf` | `/routing bgp` |

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| RIP routes not received | Version mismatch (v1 vs v2) | Set `send=v2 receive=v2` on all peers |
| RIP routes received but not installed | Route preference / distance | Check for lower-distance routes winning |
| OSPF neighbors stuck in `Init` | Mismatched area ID or hello/dead timers | Verify area IDs and timers match on both sides |
| OSPF neighbors stuck in `2-Way` | Not on broadcast network needing DR/BDR, or network type mismatch | Set correct OSPF network type (point-to-point, broadcast) |
| BGP stuck in `Active` | Peer IP or AS number wrong; firewall blocking TCP 179 | Verify peer address, AS, and firewall rules |
| BGP session flapping | MTU mismatch or route table churn | Check interface MTU; reduce BGP timers |
| Routes not redistributed | Redistribution not enabled on instance | Enable `redistribute=` setting on the protocol instance |

```bash
# Enable debug logging for RIP
/system logging add topics=rip,debug

# Enable debug logging for OSPF
/system logging add topics=ospf,debug

# Enable debug logging for BGP
/system logging add topics=bgp,debug

# View recent routing log entries
/log print where topics~"rip"
/log print where topics~"ospf"
/log print where topics~"bgp"

# View full routing table
/routing route print

# View protocol-specific routes
/routing route print where protocol=rip
/routing route print where protocol=ospf
/routing route print where protocol=bgp
```
