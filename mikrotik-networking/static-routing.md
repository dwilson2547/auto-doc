# Static Routing on MikroTik

Static routing lets you manually define paths that IP packets take through your network. It is straightforward to configure, requires no routing protocol overhead, and is ideal for small networks or for defining a default gateway on larger ones.

## Table of Contents

- [Overview](#overview)
- [Viewing the Routing Table](#viewing-the-routing-table)
- [Adding a Default Gateway](#adding-a-default-gateway)
- [Adding Static Routes](#adding-static-routes)
  - [Route to a Remote Network](#route-to-a-remote-network)
  - [Route via a Specific Interface](#route-via-a-specific-interface)
  - [Blackhole / Unreachable Routes](#blackhole--unreachable-routes)
- [Route Distance and Administrative Distance](#route-distance-and-administrative-distance)
- [Floating Static Routes (Backup Routes)](#floating-static-routes-backup-routes)
- [Recursive Routing](#recursive-routing)
- [Route Checking and Verification](#route-checking-and-verification)
- [Practical Example: Dual-WAN with Failover](#practical-example-dual-wan-with-failover)
- [WinBox / WebFig Configuration](#winbox--webfig-configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

In RouterOS, static routes are managed under `/ip route`. Each route entry specifies:

| Field | Description |
|-------|-------------|
| `dst-address` | Destination network in CIDR notation |
| `gateway` | Next-hop IP address or outgoing interface |
| `distance` | Administrative distance (lower = preferred, default 1) |
| `comment` | Optional human-readable label |

---

## Viewing the Routing Table

```bash
# Print all routes
/ip route print

# Print only active routes
/ip route print where active=yes

# Print detailed info
/ip route print detail
```

Sample output:

```
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static, r - rip, b - bgp, o - ospf
 #      DST-ADDRESS        PREF-SRC        GATEWAY            DISTANCE
 0 ADS  0.0.0.0/0                          203.0.113.1               1
 1 ADC  192.168.1.0/24    192.168.1.1      ether2                    0
 2 ADS  10.10.0.0/24                       192.168.1.254             1
```

---

## Adding a Default Gateway

A default route (`0.0.0.0/0`) matches all destinations not covered by a more specific route. Typically this points to your ISP's gateway.

```bash
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 comment="Default route - ISP1"
```

Verify:

```bash
/ip route print where dst-address=0.0.0.0/0
```

---

## Adding Static Routes

### Route to a Remote Network

Route traffic destined for `10.20.0.0/24` through a next-hop of `192.168.1.254`:

```bash
/ip route add dst-address=10.20.0.0/24 gateway=192.168.1.254 comment="Branch office network"
```

### Route via a Specific Interface

Useful for point-to-point links where no IP is assigned to the remote end:

```bash
/ip route add dst-address=172.16.5.0/24 gateway=ether3 comment="P2P link to datacenter"
```

### Blackhole / Unreachable Routes

Silently drop traffic to a destination (blackhole) or send an ICMP unreachable reply:

```bash
# Blackhole — silently drop
/ip route add dst-address=10.99.0.0/24 type=blackhole comment="Block test network"

# Unreachable — send ICMP unreachable
/ip route add dst-address=10.98.0.0/24 type=unreachable comment="Unreachable network"

# Prohibit — send ICMP administratively prohibited
/ip route add dst-address=10.97.0.0/24 type=prohibit
```

---

## Route Distance and Administrative Distance

RouterOS selects the best route to a destination based on the **distance** value. Lower distance wins.

| Route Source | Default Distance |
|-------------|-----------------|
| Connected   | 0               |
| Static      | 1               |
| eBGP        | 20              |
| OSPF        | 110             |
| RIP         | 120             |
| iBGP        | 200             |

Override the default distance:

```bash
/ip route add dst-address=10.30.0.0/24 gateway=192.168.1.254 distance=5 comment="Preferred path"
/ip route add dst-address=10.30.0.0/24 gateway=192.168.2.254 distance=10 comment="Backup path"
```

---

## Floating Static Routes (Backup Routes)

Create a primary and a backup default route using different distance values. RouterOS activates the backup only when the primary becomes unreachable.

```bash
# Primary default route via ISP1 (distance=1)
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 comment="ISP1 primary"

# Backup default route via ISP2 (distance=10, activated only if ISP1 route is gone)
/ip route add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=10 comment="ISP2 backup"
```

> **Note:** For automatic failover based on reachability (not just route presence), use the `check-gateway` option or combine with Netwatch. See [Practical Example](#practical-example-dual-wan-with-failover) below.

---

## Recursive Routing

A recursive route resolves the gateway via another route. This is useful when the next-hop is not directly connected.

```bash
# This resolves the gateway 10.0.0.1 using the routing table
/ip route add dst-address=172.16.100.0/24 gateway=10.0.0.1
```

RouterOS will perform a recursive lookup to find the egress interface for `10.0.0.1`.

---

## Route Checking and Verification

Use `check-gateway` to have RouterOS actively monitor a route's gateway and remove the route if the gateway becomes unreachable.

```bash
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 check-gateway=ping distance=1 comment="ISP1 with liveness check"
```

Options for `check-gateway`:

| Value | Description |
|-------|-------------|
| `ping` | Sends ICMP pings to the gateway |
| `arp`  | Uses ARP to verify the gateway is reachable |

---

## Practical Example: Dual-WAN with Failover

**Topology:**
- `ether1` → ISP1 — gateway `203.0.113.1`
- `ether2` → ISP2 — gateway `198.51.100.1`
- LAN on `ether3` — `192.168.1.0/24`

```bash
# ISP1 interfaces (assume already configured with IP addresses)
# /ip address already has 203.0.113.2/30 on ether1 and 198.51.100.2/30 on ether2

# Primary default route with gateway liveness check
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=203.0.113.1 \
    distance=1 \
    check-gateway=ping \
    comment="ISP1 primary"

# Backup default route
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=198.51.100.1 \
    distance=10 \
    check-gateway=ping \
    comment="ISP2 backup"

# Source NAT for both ISPs
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="NAT via ISP1"

/ip firewall nat add \
    chain=srcnat \
    out-interface=ether2 \
    action=masquerade \
    comment="NAT via ISP2"
```

When `203.0.113.1` stops responding to ping, RouterOS removes the primary route and the backup route (`distance=10`) becomes active automatically.

---

## WinBox / WebFig Configuration

1. Open **WinBox** → **IP** → **Routes**
2. Click the **+** (Add) button
3. Fill in:
   - **Dst. Address**: `0.0.0.0/0` (or your target network)
   - **Gateway**: next-hop IP or interface name
   - **Distance**: administrative distance
4. Click **OK**

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Route present but not active (`A` flag missing) | Gateway unreachable or duplicate route with lower distance | Check gateway connectivity; check for conflicting routes |
| Traffic not following expected route | More specific route exists elsewhere | Run `/ip route print` and look for more specific (`/32`, `/30`) entries |
| Route disappears intermittently | `check-gateway` failing | Verify gateway responds to ICMP/ARP; check firewall rules |
| No internet access despite default route | Missing NAT masquerade rule | Add `/ip firewall nat` masquerade on WAN interface |

```bash
# Test if a specific destination is reachable
/tool traceroute 8.8.8.8

# Look up which route would be used for a destination
/ip route print where dst-address in 8.8.8.8/32
```
