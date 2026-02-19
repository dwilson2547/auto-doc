# Trunk and Access Port Configuration on MikroTik Switches

This guide covers configuring trunk (tagged) and access (untagged) ports on MikroTik CRS (Cloud Router Switch) and CSS (Cloud Smart Switch) devices. It builds on the bridge VLAN filtering approach described in the [VLANs guide](vlans.md).

## Table of Contents

- [Overview](#overview)
- [Trunk vs. Access Ports](#trunk-vs-access-ports)
- [Bridge VLAN Filtering Model](#bridge-vlan-filtering-model)
- [Access Port Configuration](#access-port-configuration)
- [Trunk Port Configuration](#trunk-port-configuration)
- [Hybrid Ports (Tagged + Untagged)](#hybrid-ports-tagged--untagged)
- [Native VLAN on a Trunk Port](#native-vlan-on-a-trunk-port)
- [Complete Switch Example](#complete-switch-example)
  - [Topology](#topology)
  - [Full Configuration](#full-configuration)
- [Hardware Offload on CRS Switches](#hardware-offload-on-crs-switches)
- [Verifying Port Configuration](#verifying-port-configuration)
- [WinBox / WebFig Configuration](#winbox--webfig-configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

In MikroTik RouterOS, all switching is handled via **bridge** interfaces. Trunk and access port behavior is defined by:

1. **Bridge VLAN table** — which VLANs are allowed on each port, and whether frames are tagged or untagged
2. **Port PVID (Port VLAN ID)** — the VLAN assigned to untagged ingress frames

> This guide applies to CRS1xx, CRS2xx, CRS3xx, and CSS series switches running RouterOS or SwOS.

---

## Trunk vs. Access Ports

| Feature | Access Port | Trunk Port |
|---------|------------|-----------|
| VLANs carried | One VLAN only | Multiple VLANs |
| Frames on the wire | **Untagged** (no 802.1Q header) | **Tagged** (802.1Q header present) |
| Connected to | End devices (PCs, printers, phones) | Other switches, routers, APs |
| PVID | Set to the access VLAN | Usually set to the native VLAN (e.g., 1) |
| VLAN table entry | Listed as `untagged` | Listed as `tagged` |

---

## Bridge VLAN Filtering Model

RouterOS uses a centralized **bridge VLAN table** rather than per-port VLAN lists. Each VLAN entry specifies:

- `vlan-ids` — the VLAN number(s)
- `tagged` — ports that carry this VLAN with the 802.1Q header
- `untagged` — ports that carry this VLAN without the 802.1Q header

A port appears in exactly one VLAN's `untagged` list (matching its PVID) but can appear in multiple VLANs' `tagged` lists.

---

## Access Port Configuration

An access port belongs to a single VLAN. Frames leaving the port have the VLAN tag stripped; frames arriving are assumed to belong to the port's PVID.

```bash
# Step 1: Add the port to the bridge
/interface bridge port add \
    interface=ether3 \
    bridge=bridge1 \
    pvid=20 \
    comment="Access port — VLAN 20 (Servers)"

# Step 2: Add the port as untagged in the VLAN table
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=20 \
    untagged=ether3

# OR if the VLAN entry already exists, edit it to add the port
/interface bridge vlan set [find vlan-ids=20] untagged=ether3,ether4
```

> The `pvid` and the VLAN in the `untagged` list **must match**. A mismatch will cause frames to be dropped.

---

## Trunk Port Configuration

A trunk port carries multiple VLANs with 802.1Q tags. Typically used for uplinks to routers, other switches, or wireless access points that support multiple SSIDs on separate VLANs.

```bash
# Step 1: Add the uplink port to the bridge (pvid=1 for native VLAN)
/interface bridge port add \
    interface=ether1 \
    bridge=bridge1 \
    pvid=1 \
    comment="Trunk uplink to router"

# Step 2: Add the port as tagged to each VLAN it should carry
/interface bridge vlan set [find vlan-ids=10] tagged=ether1
/interface bridge vlan set [find vlan-ids=20] tagged=ether1
/interface bridge vlan set [find vlan-ids=30] tagged=ether1
```

Or when creating new VLAN entries for all VLANs at once:

```bash
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=ether1 untagged=ether2
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=ether1 untagged=ether3
/interface bridge vlan add bridge=bridge1 vlan-ids=30 tagged=ether1 untagged=ether4,ether5
```

---

## Hybrid Ports (Tagged + Untagged)

A hybrid port carries one untagged VLAN (voice or data) and additional tagged VLANs. Common for IP phones that pass through a PC on the same port.

Example: `ether6` is a phone/PC port — untagged VLAN 30 (data) + tagged VLAN 20 (voice):

```bash
# Port PVID = 30 (data VLAN for the PC)
/interface bridge port set [find interface=ether6] pvid=30

# VLAN 30: ether6 is untagged (PC traffic)
/interface bridge vlan set [find vlan-ids=30] untagged=ether6

# VLAN 20: ether6 is tagged (phone traffic)
/interface bridge vlan set [find vlan-ids=20] tagged=ether6
```

The IP phone tags its own traffic with VLAN 20; the PC sends untagged traffic which is assigned to VLAN 30.

---

## Native VLAN on a Trunk Port

The native VLAN is the VLAN used for untagged frames on a trunk port. In a multi-vendor environment, ensure the native VLAN matches on both ends.

```bash
# Set native VLAN to VLAN 99 on trunk port ether1
/interface bridge port set [find interface=ether1] pvid=99

# VLAN 99 must list ether1 as untagged in the VLAN table
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=99 \
    tagged=bridge1 \
    untagged=ether1

# All other VLANs remain tagged
/interface bridge vlan set [find vlan-ids=10] tagged=ether1
/interface bridge vlan set [find vlan-ids=20] tagged=ether1
```

> **Best practice:** Avoid using VLAN 1 as the native VLAN. Use a dedicated, unused VLAN (e.g., VLAN 99) for native traffic or disable native VLAN traffic entirely.

---

## Complete Switch Example

### Topology

```
                    ┌──────────────────────────────────┐
                    │     MikroTik CRS326              │
                    │                                  │
 ether1 ─── [Router] (Trunk: VLANs 10,20,30)         │
 ether2 ─── [AP]     (Trunk: VLANs 30,40)            │
 ether3 ─── [Server] (Access: VLAN 20)               │
 ether4 ─── [PC]     (Access: VLAN 30)               │
 ether5 ─── [PC]     (Access: VLAN 30)               │
 ether6 ─── [Phone]  (Access: VLAN 10)               │
 ether7 ─── [Guest]  (Access: VLAN 40)               │
                    └──────────────────────────────────┘

VLAN 10 = Management (192.168.10.0/24)
VLAN 20 = Servers    (192.168.20.0/24)
VLAN 30 = Clients    (192.168.30.0/24)
VLAN 40 = Guest      (192.168.40.0/24)
```

### Full Configuration

```bash
# Create bridge with VLAN filtering disabled (enable at the end)
/interface bridge add \
    name=bridge1 \
    vlan-filtering=no \
    protocol-mode=rstp \
    comment="Main switch bridge"

# Add all ports to bridge
/interface bridge port add interface=ether1 bridge=bridge1 pvid=1    # Trunk to router
/interface bridge port add interface=ether2 bridge=bridge1 pvid=1    # Trunk to AP
/interface bridge port add interface=ether3 bridge=bridge1 pvid=20   # Server - VLAN 20
/interface bridge port add interface=ether4 bridge=bridge1 pvid=30   # PC - VLAN 30
/interface bridge port add interface=ether5 bridge=bridge1 pvid=30   # PC - VLAN 30
/interface bridge port add interface=ether6 bridge=bridge1 pvid=10   # Phone - VLAN 10
/interface bridge port add interface=ether7 bridge=bridge1 pvid=40   # Guest - VLAN 40

# Configure edge ports (end devices - faster STP convergence)
/interface bridge port set [find interface=ether3] edge=yes bpdu-guard=yes
/interface bridge port set [find interface=ether4] edge=yes bpdu-guard=yes
/interface bridge port set [find interface=ether5] edge=yes bpdu-guard=yes
/interface bridge port set [find interface=ether6] edge=yes bpdu-guard=yes
/interface bridge port set [find interface=ether7] edge=yes bpdu-guard=yes

# Define VLAN table
# VLAN 10 — Management
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=10 \
    tagged=ether1 \
    untagged=ether6

# VLAN 20 — Servers
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=20 \
    tagged=ether1 \
    untagged=ether3

# VLAN 30 — Clients
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=30 \
    tagged=ether1,ether2 \
    untagged=ether4,ether5

# VLAN 40 — Guest
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=40 \
    tagged=ether1,ether2 \
    untagged=ether7

# Enable VLAN filtering (do this last to avoid lockout)
/interface bridge set bridge1 vlan-filtering=yes
```

---

## Hardware Offload on CRS Switches

CRS3xx series switches support full hardware offload of bridge/VLAN forwarding, which significantly increases throughput:

```bash
# Enable hardware offload on bridge ports
/interface bridge port set [find bridge=bridge1] hw=yes

# Verify hardware offload is active
/interface bridge port print where hw=yes
```

> Hardware offload requires `vlan-filtering=yes` on the bridge and is not available on CRS1xx/CRS2xx series.

Check the switch chip status:

```bash
/interface ethernet switch print
/interface ethernet switch port print
```

---

## Verifying Port Configuration

```bash
# Show all bridge ports with PVID and edge settings
/interface bridge port print

# Show VLAN table
/interface bridge vlan print

# Verify a specific VLAN's tagged/untagged ports
/interface bridge vlan print where vlan-ids=30

# Check MAC address table (forwarding database)
/interface bridge host print

# Show MAC table for a specific VLAN
/interface bridge host print where vid=30
```

---

## WinBox / WebFig Configuration

### Adding an Access Port

1. **Bridge** → **Ports** tab → Click **+**
2. Select **Interface** (e.g., `ether4`) and **Bridge** (`bridge1`)
3. Set **PVID** to the access VLAN (e.g., `30`)
4. Click **OK**
5. **Bridge** → **VLANs** tab → Find or create VLAN 30 → Add `ether4` to **Untagged** list

### Adding a Trunk Port

1. **Bridge** → **Ports** tab → Click **+**
2. Select **Interface** (uplink port) and **Bridge**
3. Set **PVID** to `1` (or native VLAN)
4. Click **OK**
5. For each VLAN → **Bridge** → **VLANs** → Add the trunk port to the **Tagged** list

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Access port traffic not reaching VLAN | PVID not matching VLAN table untagged entry | Set port PVID to match VLAN and add as `untagged` in VLAN table |
| Trunk not passing all VLANs | VLANs not listed as `tagged` on trunk port | Add trunk port to `tagged=` list for each VLAN |
| Hosts in same VLAN cannot see each other | VLAN filtering enabled but VLAN entry missing | Add or verify VLAN entry in bridge VLAN table |
| Flooding across all ports | VLAN filtering not enabled on bridge | Set `vlan-filtering=yes` on bridge |
| Management access lost after VLAN changes | Bridge VLAN table excludes management VLAN | Ensure management VLAN includes the management port as tagged or untagged |

```bash
# Quick check: does VLAN 30 include the expected ports?
/interface bridge vlan print where vlan-ids=30

# Is VLAN filtering actually enabled?
/interface bridge print where name=bridge1

# Check port PVID
/interface bridge port print detail where interface=ether4
```
