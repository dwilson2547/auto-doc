# Spanning Tree Protocol on MikroTik

Spanning Tree Protocol (STP) prevents Layer 2 loops in networks with redundant paths between switches. MikroTik RouterOS supports STP, RSTP (Rapid STP), and MSTP (Multiple Spanning Tree Protocol) through its bridge implementation.

## Table of Contents

- [Overview](#overview)
- [STP Variants](#stp-variants)
- [How STP Works](#how-stp-works)
  - [Root Bridge Election](#root-bridge-election)
  - [Port States and Roles](#port-states-and-roles)
- [Configuring STP on a MikroTik Bridge](#configuring-stp-on-a-mikrotik-bridge)
  - [Enable STP / RSTP](#enable-stp--rstp)
  - [Set Bridge Priority (Root Bridge Election)](#set-bridge-priority-root-bridge-election)
  - [Set Port Path Cost](#set-port-path-cost)
  - [Set Port Priority](#set-port-priority)
- [RSTP Configuration](#rstp-configuration)
- [MSTP Configuration](#mstp-configuration)
- [PortFast / Edge Ports](#portfast--edge-ports)
- [BPDU Guard](#bpdu-guard)
- [Monitoring Spanning Tree](#monitoring-spanning-tree)
- [Multi-Switch STP Example](#multi-switch-stp-example)
- [Troubleshooting](#troubleshooting)

---

## Overview

Without STP, a network with redundant switch paths will create a **broadcast storm** as frames loop indefinitely. STP solves this by:

1. Electing a **Root Bridge** — the central reference switch
2. Calculating shortest paths to the root
3. **Blocking** redundant ports to prevent loops
4. **Unblocking** those ports automatically if the active path fails

---

## STP Variants

| Protocol | Standard | Convergence Time | Notes |
|----------|----------|-----------------|-------|
| **STP** | IEEE 802.1D (1998) | 30–50 seconds | Legacy; rarely used today |
| **RSTP** | IEEE 802.1w / 802.1D-2004 | 1–6 seconds | Recommended for most networks |
| **MSTP** | IEEE 802.1s / 802.1Q-2005 | 1–6 seconds | Multiple spanning tree instances; useful with VLANs |

RouterOS supports all three via the `protocol-mode` setting on a bridge interface.

---

## How STP Works

### Root Bridge Election

Every bridge has a **Bridge ID** composed of:
- **Priority** (0–61440, in increments of 4096; default 32768)
- **MAC address**

The bridge with the **lowest Bridge ID** becomes the root bridge. In case of a tie in priority, the lower MAC address wins. You control root bridge placement by setting a lower priority.

### Port States and Roles

**STP port states:**

| State | Description |
|-------|-------------|
| **Disabled** | Administratively down |
| **Blocking** | Receives BPDUs but does not forward data frames |
| **Listening** | Processing BPDUs; not forwarding data |
| **Learning** | Building MAC table; not forwarding data |
| **Forwarding** | Normal data forwarding |

**RSTP port roles:**

| Role | Description |
|------|-------------|
| **Root Port** | Best path toward the root bridge |
| **Designated Port** | Forwards traffic away from the root on a segment |
| **Alternate Port** | Backup root port (blocked) |
| **Backup Port** | Backup designated port (blocked) |
| **Edge Port** | Connected to an end device (no BPDU expected) |

---

## Configuring STP on a MikroTik Bridge

### Enable STP / RSTP

STP is configured per bridge interface:

```bash
# Enable RSTP on bridge1
/interface bridge set bridge1 \
    protocol-mode=rstp \
    priority=0x8000 \
    forward-delay=15 \
    max-message-age=20 \
    hello-time=2
```

| Parameter | Description | Default |
|-----------|-------------|---------|
| `protocol-mode` | `none`, `stp`, `rstp`, or `mstp` | `rstp` |
| `priority` | Bridge priority (0–61440, multiples of 4096) | `32768` (0x8000) |
| `forward-delay` | Time (seconds) spent in Listening/Learning states | `15` |
| `max-message-age` | Maximum age of received BPDUs before topology change | `20` |
| `hello-time` | Interval between BPDU transmissions | `2` |

### Set Bridge Priority (Root Bridge Election)

Set a lower priority to make a bridge the preferred root:

```bash
# Make this bridge the root bridge (lowest priority wins)
/interface bridge set bridge1 priority=0x1000   # decimal: 4096

# Make this bridge the backup root
/interface bridge set bridge2 priority=0x2000   # decimal: 8192
```

> **Tip:** Use multiples of 4096. Common values: `4096` (root), `8192` (secondary), `32768` (default).

### Set Port Path Cost

The path cost determines which path is preferred toward the root bridge. Lower cost = preferred path.

```bash
# Set path cost for a specific bridge port
/interface bridge port set [find interface=ether2] path-cost=10

# Default costs by link speed (802.1D-1998):
# 10 Mbps  = 100
# 100 Mbps = 19
# 1 Gbps   = 4
# 10 Gbps  = 2
```

RSTP long path cost (802.1D-2004):

```bash
/interface bridge port set [find interface=ether2] path-cost=20000  # for 1G uplink
```

Enable long path cost mode for the bridge:

```bash
/interface bridge set bridge1 region-name=default
```

### Set Port Priority

Port priority is used as a tiebreaker when two ports on the same bridge have equal path cost to the root. Lower priority = preferred.

```bash
# Set port priority (0–240, in multiples of 16; default 128)
/interface bridge port set [find interface=ether3] priority=64
```

---

## RSTP Configuration

RSTP is the default and recommended mode. Ports with RSTP converge faster because they negotiate directly with their neighbor rather than waiting for timers.

```bash
/interface bridge set bridge1 protocol-mode=rstp
```

For RSTP to work optimally:
- Set **edge ports** on all ports connected to end devices (see [PortFast / Edge Ports](#portfast--edge-ports))
- Ensure only network equipment receives BPDUs on trunk ports
- Use BPDU Guard on edge ports (see [BPDU Guard](#bpdu-guard))

---

## MSTP Configuration

MSTP allows different VLANs to follow different spanning tree instances, enabling per-VLAN load balancing.

```bash
# Enable MSTP on the bridge
/interface bridge set bridge1 \
    protocol-mode=mstp \
    region-name="MY_REGION" \
    region-revision=1
```

### Define MSTP Instances

```bash
# Create instance 1 for VLANs 10 and 20
/interface bridge msti add \
    bridge=bridge1 \
    identifier=1 \
    priority=0x1000

# Map VLANs to MSTP instances
/interface bridge vlan set [find vlan-ids=10] msti=1
/interface bridge vlan set [find vlan-ids=20] msti=1

# Create instance 2 for VLANs 30 and 40
/interface bridge msti add \
    bridge=bridge1 \
    identifier=2 \
    priority=0x2000

/interface bridge vlan set [find vlan-ids=30] msti=2
/interface bridge vlan set [find vlan-ids=40] msti=2
```

> All switches participating in MSTP must have the **same region name and revision** to belong to the same region.

---

## PortFast / Edge Ports

Mark ports connected to end devices (PCs, servers, phones) as **edge ports**. Edge ports skip the Listening and Learning states and go directly to Forwarding, eliminating the 15–30 second delay for end devices.

```bash
# Mark a port as edge
/interface bridge port set [find interface=ether4] edge=yes

# Or set edge=auto (RSTP auto-detects edge ports — no BPDUs received)
/interface bridge port set [find interface=ether4] edge=auto
```

> **Never** set `edge=yes` on ports connected to other switches, as this can create loops.

---

## BPDU Guard

BPDU Guard shuts down a port if a BPDU is received on it. This protects edge ports from rogue switches or misconfigured devices being plugged in.

```bash
# Enable BPDU Guard on edge ports
/interface bridge port set [find interface=ether4] bpdu-guard=yes
/interface bridge port set [find interface=ether5] bpdu-guard=yes
```

If a BPDU is received on a BPDU Guard–protected port, RouterOS will **disable** the port and log the event. To re-enable it after investigating:

```bash
/interface bridge port set [find interface=ether4] disabled=no
```

View BPDU Guard events:

```bash
/log print where message~"bpdu-guard"
```

---

## Monitoring Spanning Tree

```bash
# View bridge STP status
/interface bridge print detail

# View port roles and states
/interface bridge port print

# Detailed per-port STP information
/interface bridge port print detail

# Monitor topology changes
/log print where topics~"bridge"

# Check root bridge status
/interface bridge monitor bridge1
```

Sample output of `/interface bridge monitor bridge1`:

```
          state: enabled
  current-mac-address: DC:2C:6E:AA:BB:CC
          root-bridge: yes
   root-bridge-id: 0x1000.DC:2C:6E:AA:BB:CC
         root-path-cost: 0
              root-port:
     port-count: 4
  designated-port-count: 4
```

---

## Multi-Switch STP Example

**Topology:** Three switches with redundant uplinks.

```
          [Root Switch — SW1]
          priority=4096
         /               \
  [SW2 priority=8192]   [SW3 priority=8192]
         \               /
          [Access Switch]
```

**SW1 (Root Bridge) configuration:**

```bash
/interface bridge set bridge1 protocol-mode=rstp priority=0x1000
# All ports are designated ports toward root
```

**SW2 (Secondary root) configuration:**

```bash
/interface bridge set bridge1 protocol-mode=rstp priority=0x2000

# Uplink to SW1 on ether1 (root port — preferred path)
/interface bridge port set [find interface=ether1] path-cost=4

# Uplink to SW3 on ether2 (alternate port — blocked if SW1 uplink is up)
/interface bridge port set [find interface=ether2] path-cost=8

# Access ports — edge ports for hosts
/interface bridge port set [find interface=ether3] edge=yes bpdu-guard=yes
/interface bridge port set [find interface=ether4] edge=yes bpdu-guard=yes
```

**SW3 configuration:**

```bash
/interface bridge set bridge1 protocol-mode=rstp priority=0x3000
/interface bridge port set [find interface=ether1] path-cost=4   # uplink to SW1
/interface bridge port set [find interface=ether2] path-cost=8   # uplink to SW2
```

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Broadcast storm / high CPU on switches | STP disabled or loop created | Enable STP/RSTP on all bridges; check for unmanaged switches |
| Slow convergence (30+ seconds) | Using STP instead of RSTP, or edge ports not configured | Switch to RSTP; mark end-device ports as `edge=yes` |
| Unexpected root bridge | Bridge with lower MAC or accidental lower priority | Set explicit priorities; use 4096 for intended root |
| Port blocked unexpectedly | STP elected a different root; path cost imbalance | Set lower path cost on preferred uplinks |
| BPDU Guard disabled port | Rogue switch or misconfigured device plugged in | Investigate the connected device; re-enable port after fixing |

```bash
# Check which bridge port is the root port
/interface bridge port print where role=root-port

# Check for blocked ports
/interface bridge port print where status=inactive

# Print STP events
/log print topics~"bridge" follow
```
