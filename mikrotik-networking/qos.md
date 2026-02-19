# Quality of Service (QoS) on MikroTik

Quality of Service (QoS) allows you to prioritize, rate-limit, and shape network traffic so that critical applications (voice, video, management) receive the bandwidth and low latency they need, even under congestion. MikroTik RouterOS provides several QoS mechanisms: Simple Queues, Queue Trees, and Mangle-based packet marking.

## Table of Contents

- [Overview](#overview)
- [QoS Concepts](#qos-concepts)
- [Simple Queues](#simple-queues)
  - [Limit a Host's Download and Upload](#limit-a-hosts-download-and-upload)
  - [Burst Behavior](#burst-behavior)
  - [Queue for an Entire Subnet](#queue-for-an-entire-subnet)
- [Queue Types (PCQ, SFQ, RED, FIFO)](#queue-types-pcq-sfq-red-fifo)
  - [PCQ for Per-Client Fair Bandwidth Sharing](#pcq-for-per-client-fair-bandwidth-sharing)
- [Mangle — Traffic Marking](#mangle--traffic-marking)
  - [Mark VoIP Traffic](#mark-voip-traffic)
  - [Mark Video Streaming Traffic](#mark-video-streaming-traffic)
  - [Mark by DSCP / TOS](#mark-by-dscp--tos)
- [Queue Trees — Hierarchical QoS](#queue-trees--hierarchical-qos)
  - [Two-Level Hierarchy Example](#two-level-hierarchy-example)
  - [Full Prioritization Example (VoIP + Video + Data)](#full-prioritization-example-voip--video--data)
- [DSCP Marking and Remarking](#dscp-marking-and-remarking)
- [HTB vs. HFSC](#htb-vs-hfsc)
- [WinBox / WebFig Configuration](#winbox--webfig-configuration)
- [Verifying QoS](#verifying-qos)
- [Troubleshooting](#troubleshooting)

---

## Overview

RouterOS QoS operates on traffic leaving an interface (egress). Key components:

| Component | Purpose |
|-----------|---------|
| **Simple Queue** | Straightforward per-IP or per-subnet rate limiting |
| **Queue Tree** | Hierarchical QoS with traffic prioritization |
| **Mangle** | Mark packets for later processing by queues |
| **Queue Types** | Algorithms controlling how queued packets are scheduled (FIFO, SFQ, PCQ, RED) |

> **Important:** RouterOS queues control **upload** (traffic leaving an interface). To shape **download**, use an intermediate queue on the LAN-side interface.

---

## QoS Concepts

| Term | Description |
|------|-------------|
| **Rate limit (max-limit)** | Hard ceiling on bandwidth |
| **Guaranteed rate (limit-at)** | Minimum guaranteed bandwidth |
| **Priority** | 1 (highest) – 8 (lowest); within the same parent queue |
| **Burst** | Allow exceeding max-limit for a short burst period |
| **HTB** | Hierarchical Token Bucket — the default scheduling algorithm |
| **DSCP** | Differentiated Services Code Point — 6-bit QoS marking in IP header |

---

## Simple Queues

Simple Queues are the easiest way to apply per-IP rate limits. They are evaluated in order (first match wins).

### Limit a Host's Download and Upload

```bash
/queue simple add \
    name="Limit-PC1" \
    target=192.168.1.100/32 \
    max-limit=20M/10M \
    comment="PC1 — 20Mbit down / 10Mbit up"
```

The format for `max-limit` is `download/upload` (from the perspective of the target host).

### Burst Behavior

Allow a host to burst above its limit for a short period:

```bash
/queue simple add \
    name="Burst-PC2" \
    target=192.168.1.101/32 \
    max-limit=20M/10M \
    burst-limit=40M/20M \
    burst-threshold=10M/5M \
    burst-time=8s/8s \
    comment="PC2 — burst to 40M for 8 seconds"
```

| Parameter | Description |
|-----------|-------------|
| `burst-limit` | Maximum speed during burst period |
| `burst-threshold` | When average speed drops below this, burst is allowed |
| `burst-time` | Duration (seconds) that burst limit is averaged over |

### Queue for an Entire Subnet

```bash
/queue simple add \
    name="Limit-Guests" \
    target=192.168.40.0/24 \
    max-limit=50M/25M \
    comment="Guest VLAN — total 50M down / 25M up"
```

---

## Queue Types (PCQ, SFQ, RED, FIFO)

Queue types define the packet scheduling algorithm used inside a queue.

| Type | Description | Best For |
|------|-------------|----------|
| **FIFO** | First In, First Out — simple, no fairness | Low-traffic links |
| **SFQ** | Stochastic Fairness Queuing — randomized fairness between flows | General fair queuing |
| **RED** | Random Early Detection — drops packets early to avoid congestion | High-speed links |
| **PCQ** | Per-Connection Queuing — divides bandwidth equally per flow | Per-user fair sharing on ISP links |

### PCQ for Per-Client Fair Bandwidth Sharing

PCQ divides total bandwidth equally among all active flows/clients:

```bash
# Create download PCQ type
/queue type add \
    name="pcq-download" \
    kind=pcq \
    pcq-classifier=dst-address \
    pcq-rate=0 \
    pcq-limit=50KiB \
    pcq-total-limit=2000KiB

# Create upload PCQ type
/queue type add \
    name="pcq-upload" \
    kind=pcq \
    pcq-classifier=src-address \
    pcq-rate=0 \
    pcq-limit=50KiB \
    pcq-total-limit=2000KiB

# Apply PCQ in a simple queue
/queue simple add \
    name="PCQ-LAN" \
    target=192.168.1.0/24 \
    queue=pcq-upload/pcq-download \
    max-limit=100M/100M \
    comment="Fair PCQ for all LAN clients"
```

---

## Mangle — Traffic Marking

Mangle marks packets with connection marks or packet marks that Queue Trees use to apply QoS policies.

> **Always mark the connection first, then mark packets based on the connection mark.** This is more efficient than marking every individual packet.

### Mark VoIP Traffic

```bash
# Mark VoIP connections (SIP + RTP)
/ip firewall mangle add \
    chain=prerouting \
    protocol=udp \
    dst-port=5060 \
    action=mark-connection \
    new-connection-mark=voip-conn \
    passthrough=yes \
    comment="Mark SIP connections"

/ip firewall mangle add \
    chain=prerouting \
    protocol=udp \
    dst-port=10000-20000 \
    action=mark-connection \
    new-connection-mark=voip-conn \
    passthrough=yes \
    comment="Mark RTP connections"

# Mark all packets belonging to VoIP connections
/ip firewall mangle add \
    chain=prerouting \
    connection-mark=voip-conn \
    action=mark-packet \
    new-packet-mark=voip-pkt \
    passthrough=no \
    comment="Mark VoIP packets"
```

### Mark Video Streaming Traffic

```bash
# Mark Netflix, YouTube, etc. by DSCP AF41 (video streaming)
/ip firewall mangle add \
    chain=prerouting \
    dscp=18 \
    action=mark-connection \
    new-connection-mark=video-conn \
    passthrough=yes

/ip firewall mangle add \
    chain=prerouting \
    connection-mark=video-conn \
    action=mark-packet \
    new-packet-mark=video-pkt \
    passthrough=no
```

### Mark by DSCP / TOS

```bash
# Mark traffic already marked CS5 (signaling) or EF (expedited forwarding)
/ip firewall mangle add \
    chain=prerouting \
    dscp=46 \
    action=mark-packet \
    new-packet-mark=voip-ef \
    passthrough=no \
    comment="EF DSCP — typically VoIP media"
```

---

## Queue Trees — Hierarchical QoS

Queue Trees provide multi-level QoS with guaranteed rates and priorities. They attach to an interface's `global-out` (egress) or `global-in` (ingress) queue.

### Two-Level Hierarchy Example

```
[global-out]
    └── [WAN-upload 100M]
            ├── [VoIP: priority=1, guaranteed 2M]
            ├── [Video: priority=2, guaranteed 10M]
            └── [Data: priority=3, best-effort]
```

```bash
# Parent queue — total WAN upload bandwidth
/queue tree add \
    name="WAN-upload" \
    parent=global-out \
    max-limit=100M \
    comment="Total WAN upload shaper"

# VoIP — highest priority, guaranteed 2M
/queue tree add \
    name="voip-up" \
    parent=WAN-upload \
    packet-mark=voip-pkt \
    priority=1 \
    limit-at=2M \
    max-limit=10M \
    comment="VoIP upload — priority 1"

# Video — second priority
/queue tree add \
    name="video-up" \
    parent=WAN-upload \
    packet-mark=video-pkt \
    priority=2 \
    limit-at=10M \
    max-limit=50M \
    comment="Video upload — priority 2"

# Data — bulk traffic
/queue tree add \
    name="data-up" \
    parent=WAN-upload \
    packet-mark="" \
    priority=8 \
    limit-at=5M \
    max-limit=100M \
    comment="Bulk data — lowest priority"
```

### Full Prioritization Example (VoIP + Video + Data)

This example controls both upload and download:

```bash
# --- Mangle rules (see Mangle section above) ---

# --- Parent queues ---
/queue tree add name="WAN-out" parent=ether1 max-limit=100M
/queue tree add name="LAN-out" parent=bridge1 max-limit=1000M

# --- WAN upload (ether1 egress) priority queues ---
/queue tree add name="voip-wan-up"  parent=WAN-out packet-mark=voip-pkt  priority=1 limit-at=2M  max-limit=20M
/queue tree add name="video-wan-up" parent=WAN-out packet-mark=video-pkt priority=3 limit-at=10M max-limit=60M
/queue tree add name="data-wan-up"  parent=WAN-out packet-mark=""         priority=8 limit-at=5M  max-limit=100M

# --- LAN download (bridge1 egress) priority queues ---
/queue tree add name="voip-lan-dn"  parent=LAN-out packet-mark=voip-pkt  priority=1 limit-at=2M   max-limit=20M
/queue tree add name="video-lan-dn" parent=LAN-out packet-mark=video-pkt priority=3 limit-at=20M  max-limit=200M
/queue tree add name="data-lan-dn"  parent=LAN-out packet-mark=""         priority=8 limit-at=10M max-limit=1000M
```

---

## DSCP Marking and Remarking

Set or remark DSCP values on outgoing packets to comply with upstream QoS policies:

```bash
# Set DSCP EF (46) on VoIP packets leaving WAN
/ip firewall mangle add \
    chain=postrouting \
    packet-mark=voip-pkt \
    out-interface=ether1 \
    action=change-dscp \
    new-dscp=46 \
    comment="Remark VoIP as DSCP EF toward ISP"

# Set DSCP AF41 (34) on video packets
/ip firewall mangle add \
    chain=postrouting \
    packet-mark=video-pkt \
    out-interface=ether1 \
    action=change-dscp \
    new-dscp=34 \
    comment="Remark video as DSCP AF41 toward ISP"
```

Common DSCP values:

| DSCP Name | Value | Decimal | Use Case |
|-----------|-------|---------|---------|
| Default (BE) | 000000 | 0 | Best effort |
| AF11 | 001010 | 10 | Low-priority data |
| AF21 | 010010 | 18 | Video |
| AF31 | 011010 | 26 | Transactional data |
| AF41 | 100010 | 34 | Broadcast video |
| CS5 | 101000 | 40 | Signaling |
| EF | 101110 | 46 | VoIP (expedited forwarding) |
| CS6 | 110000 | 48 | Network control |

---

## HTB vs. HFSC

RouterOS Queue Trees use **HTB (Hierarchical Token Bucket)** by default, which is appropriate for most deployments.

| Algorithm | Description | Use When |
|-----------|-------------|----------|
| **HTB** | Token-based hierarchical shaping | General-purpose rate limiting and prioritization |
| **HFSC** | Hierarchical Fair Service Curve | Strict delay guarantees needed (advanced VoIP/realtime) |

HFSC is available in RouterOS v7 and provides better latency control for real-time traffic but is more complex to configure.

---

## WinBox / WebFig Configuration

### Simple Queue

1. **Queues** → **Simple Queues** tab → Click **+**
2. Set **Name**, **Target** (IP/subnet), and **Max Limit** (`down/up`)
3. Click **OK**

### Queue Tree

1. **Queues** → **Queue Tree** tab → Click **+**
2. Set **Name**, **Parent** (interface or parent queue), **Packet Mark**, **Priority**, **Max Limit**
3. Click **OK**

### Mangle

1. **IP** → **Firewall** → **Mangle** tab → Click **+**
2. Set **Chain**, match criteria, **Action** (`mark-connection` or `mark-packet`), and **New Mark** name
3. Click **OK**

---

## Verifying QoS

```bash
# Monitor simple queue statistics in real time
/queue simple print stats

# Monitor queue tree statistics
/queue tree print stats

# Check mangle rule hit counts
/ip firewall mangle print stats

# Reset all queue counters
/queue simple reset-counters-all
/queue tree reset-counters-all

# Real-time bandwidth usage per queue
/queue simple print stats interval=1
```

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Rate limit not applied | Simple queue target mismatch | Verify target IP/subnet is correct |
| QoS not affecting traffic | Mangle rules not matching | Check mangle hit counters; verify chain and match criteria |
| VoIP still getting jitter under load | VoIP not marked correctly | Check mangle marks; verify queue tree priority and parent |
| Queue tree has no traffic | Wrong parent interface | Set parent to the correct WAN or LAN interface |
| Burst not working | Burst threshold not configured | Set `burst-threshold` below `max-limit` |

```bash
# Check if VoIP packets are being marked
/ip firewall mangle print stats where new-packet-mark=voip-pkt

# Capture traffic to verify DSCP marking
/tool sniffer quick interface=ether1 ip-protocol=udp port=5060

# Check active connections with their marks
/ip firewall connection print where connection-mark=voip-conn
```
