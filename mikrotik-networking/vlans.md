# VLAN Configuration on MikroTik

VLANs (Virtual Local Area Networks) allow you to logically segment a network at Layer 2. MikroTik RouterOS supports VLANs on both its router products and on its CRS/CSS managed switches.

## Table of Contents

- [Overview](#overview)
- [VLAN Concepts](#vlan-concepts)
- [RouterOS v7 vs v6 VLAN Approach](#routeros-v7-vs-v6-vlan-approach)
- [Setting Up VLANs on a Router (Router-on-a-Stick)](#setting-up-vlans-on-a-router-router-on-a-stick)
  - [Step 1: Create a Bridge](#step-1-create-a-bridge)
  - [Step 2: Add Physical Ports to the Bridge](#step-2-add-physical-ports-to-the-bridge)
  - [Step 3: Enable VLAN Filtering on the Bridge](#step-3-enable-vlan-filtering-on-the-bridge)
  - [Step 4: Configure VLAN Table Entries](#step-4-configure-vlan-table-entries)
  - [Step 5: Add VLAN Interfaces for Routing](#step-5-add-vlan-interfaces-for-routing)
  - [Step 6: Assign IP Addresses to VLAN Interfaces](#step-6-assign-ip-addresses-to-vlan-interfaces)
- [Inter-VLAN Routing](#inter-vlan-routing)
- [VLAN on a Physical Interface (No Bridge)](#vlan-on-a-physical-interface-no-bridge)
- [CRS Switch VLAN Configuration](#crs-switch-vlan-configuration)
- [Verifying VLAN Configuration](#verifying-vlan-configuration)
- [Common VLAN Design Examples](#common-vlan-design-examples)
- [Troubleshooting](#troubleshooting)

---

## Overview

MikroTik implements VLANs using IEEE 802.1Q tagging. Tagged frames carry a 4-byte VLAN header identifying which VLAN they belong to. Untagged frames are assigned to a PVID (Port VLAN ID) by the switch or bridge.

---

## VLAN Concepts

| Term | Description |
|------|-------------|
| **VLAN ID (VID)** | Numerical identifier for a VLAN (1–4094) |
| **Tagged port** | Port that passes frames with the 802.1Q header intact (trunk port) |
| **Untagged port** | Port that strips the 802.1Q header before forwarding (access port) |
| **PVID** | Port VLAN ID — the VLAN assigned to untagged ingress traffic |
| **Bridge VLAN Filtering** | RouterOS feature that enables VLAN-aware bridging |

---

## RouterOS v7 vs v6 VLAN Approach

| Feature | RouterOS v6 | RouterOS v7 |
|---------|------------|------------|
| VLAN filtering | `/interface bridge vlan` | Same, but improved hardware offload |
| VLAN interface | `/interface vlan` on bridge | Same |
| Switch chip offload | Limited | Full hardware offload on CRS3xx series |
| Recommended approach | Bridge + VLAN filtering | Bridge + VLAN filtering (preferred) |

> **Recommendation:** Always use bridge VLAN filtering (not the legacy `/interface ethernet switch` approach) for both v6 and v7.

---

## Setting Up VLANs on a Router (Router-on-a-Stick)

This example sets up three VLANs:

| VLAN ID | Name | Subnet |
|---------|------|--------|
| 10 | Management | 192.168.10.0/24 |
| 20 | Servers | 192.168.20.0/24 |
| 30 | Clients | 192.168.30.0/24 |

Physical layout: `ether1` is the WAN port; `ether2`–`ether5` are LAN ports added to a bridge.

### Step 1: Create a Bridge

```bash
/interface bridge add \
    name=bridge1 \
    vlan-filtering=no \
    comment="Main LAN bridge"
```

> We disable `vlan-filtering` initially so we don't lock ourselves out during setup, then enable it at the end.

### Step 2: Add Physical Ports to the Bridge

```bash
/interface bridge port add interface=ether2 bridge=bridge1 pvid=1
/interface bridge port add interface=ether3 bridge=bridge1 pvid=1
/interface bridge port add interface=ether4 bridge=bridge1 pvid=1
/interface bridge port add interface=ether5 bridge=bridge1 pvid=1
```

### Step 3: Enable VLAN Filtering on the Bridge

```bash
/interface bridge set bridge1 vlan-filtering=yes
```

### Step 4: Configure VLAN Table Entries

Define which ports are tagged (trunk) or untagged (access) per VLAN.

```bash
# VLAN 10 — Management
# bridge1 itself is tagged (router needs tagged frames for inter-VLAN routing)
# ether2 is an access (untagged) port for VLAN 10
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=10 \
    tagged=bridge1 \
    untagged=ether2

# VLAN 20 — Servers
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=20 \
    tagged=bridge1 \
    untagged=ether3

# VLAN 30 — Clients
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=30 \
    tagged=bridge1 \
    untagged=ether4,ether5
```

Update port PVIDs to match their assigned access VLAN:

```bash
/interface bridge port set [find interface=ether2] pvid=10
/interface bridge port set [find interface=ether3] pvid=20
/interface bridge port set [find interface=ether4] pvid=30
/interface bridge port set [find interface=ether5] pvid=30
```

### Step 5: Add VLAN Interfaces for Routing

Create logical VLAN interfaces on top of `bridge1` for each VLAN the router needs to participate in:

```bash
/interface vlan add name=vlan10 vlan-id=10 interface=bridge1 comment="Management VLAN"
/interface vlan add name=vlan20 vlan-id=20 interface=bridge1 comment="Servers VLAN"
/interface vlan add name=vlan30 vlan-id=30 interface=bridge1 comment="Clients VLAN"
```

### Step 6: Assign IP Addresses to VLAN Interfaces

```bash
/ip address add address=192.168.10.1/24 interface=vlan10 comment="Management gateway"
/ip address add address=192.168.20.1/24 interface=vlan20 comment="Servers gateway"
/ip address add address=192.168.30.1/24 interface=vlan30 comment="Clients gateway"
```

---

## Inter-VLAN Routing

Since the router has IP addresses on each VLAN interface, traffic between VLANs is automatically routed at Layer 3. You can control which VLANs can communicate using the **firewall**:

```bash
# Allow VLAN10 (Management) to reach all VLANs
/ip firewall filter add \
    chain=forward \
    in-interface=vlan10 \
    action=accept \
    comment="Management can reach all VLANs"

# Prevent VLAN30 (Clients) from reaching VLAN20 (Servers)
/ip firewall filter add \
    chain=forward \
    in-interface=vlan30 \
    out-interface=vlan20 \
    action=drop \
    comment="Block Clients from Servers"
```

---

## VLAN on a Physical Interface (No Bridge)

For a simple trunk link between two routers, you can create VLAN sub-interfaces directly on a physical interface:

```bash
/interface vlan add name=vlan100 vlan-id=100 interface=ether1
/interface vlan add name=vlan200 vlan-id=200 interface=ether1
/ip address add address=10.100.0.1/30 interface=vlan100
/ip address add address=10.200.0.1/30 interface=vlan200
```

---

## CRS Switch VLAN Configuration

For MikroTik CRS (Cloud Router Switch) and CSS (Cloud Smart Switch) devices in **switch mode**:

```bash
# Create the bridge with hardware offload
/interface bridge add name=bridge1 vlan-filtering=yes comment="CRS bridge"

# Add all switch ports
/interface bridge port add interface=ether1 bridge=bridge1  # trunk uplink
/interface bridge port add interface=ether2 bridge=bridge1 pvid=10
/interface bridge port add interface=ether3 bridge=bridge1 pvid=20
/interface bridge port add interface=ether4 bridge=bridge1 pvid=30

# Define VLANs — ether1 is trunk, others are access
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=ether1 untagged=ether2
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=ether1 untagged=ether3
/interface bridge vlan add bridge=bridge1 vlan-ids=30 tagged=ether1 untagged=ether4

# Enable hardware offload (CRS3xx supports this)
/interface ethernet set ether1 l2mtu=9000
/interface bridge port set [find interface=ether1] hw=yes
```

---

## Verifying VLAN Configuration

```bash
# Show all bridge VLAN entries
/interface bridge vlan print

# Show bridge ports and their PVIDs
/interface bridge port print

# Check VLAN interfaces
/interface vlan print

# Check IP addresses on VLAN interfaces
/ip address print where interface~"vlan"

# Test connectivity from the router to a VLAN host
/ping 192.168.10.100 interface=vlan10 count=4
```

---

## Common VLAN Design Examples

### Office Network

| VLAN | Name | Purpose | Subnet |
|------|------|---------|--------|
| 1 | Native | Untagged management (avoid for security) | — |
| 10 | Management | Network equipment admin | 10.0.10.0/24 |
| 20 | Voice | VoIP phones | 10.0.20.0/24 |
| 30 | Wireless | Staff Wi-Fi | 10.0.30.0/24 |
| 40 | Guest | Guest Wi-Fi (isolated) | 10.0.40.0/24 |
| 50 | Servers | Internal servers | 10.0.50.0/24 |
| 100 | DMZ | Public-facing services | 10.0.100.0/24 |

### ISP Aggregation

```
WAN Router (ether1)
     |
   trunk (VLANs 100–200)
     |
  CRS switch
  /    |    \
ether2 ether3 ether4
VLAN100 VLAN200 VLAN300
Customer A  Customer B  Customer C
```

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Hosts in same VLAN cannot communicate | VLAN filtering not enabled or PVID mismatch | Check `vlan-filtering=yes` on bridge; verify PVID on ports |
| Inter-VLAN traffic not routed | Missing VLAN interface or IP address | Ensure `/interface vlan` exists and has an IP |
| Tagged frames arriving untagged | Port not listed as tagged in VLAN table | Add port to VLAN entry as `tagged=` |
| Management access lost after enabling VLAN filtering | Management IP assigned to wrong interface | Ensure management VLAN interface has IP and bridge VLAN entry is correct |

```bash
# Debug: check bridge VLAN table
/interface bridge vlan print where vlan-ids=10

# Debug: verify bridge port PVID
/interface bridge port print detail where interface=ether2
```
