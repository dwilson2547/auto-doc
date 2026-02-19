# DHCP Configuration on MikroTik

MikroTik RouterOS includes a full-featured DHCP server and DHCP client. This guide covers configuring the DHCP server for LAN segments, creating address pools, setting DHCP options, creating static leases, and configuring the DHCP client on WAN interfaces.

## Table of Contents

- [Overview](#overview)
- [DHCP Server Quick Setup (Wizard)](#dhcp-server-quick-setup-wizard)
- [Manual DHCP Server Setup](#manual-dhcp-server-setup)
  - [Step 1: Create an IP Pool](#step-1-create-an-ip-pool)
  - [Step 2: Create the DHCP Network](#step-2-create-the-dhcp-network)
  - [Step 3: Create the DHCP Server Instance](#step-3-create-the-dhcp-server-instance)
- [Multi-VLAN DHCP Server Example](#multi-vlan-dhcp-server-example)
- [DHCP Options](#dhcp-options)
  - [Common Options](#common-options)
  - [Custom DHCP Options](#custom-dhcp-options)
- [Static (Reserved) Leases](#static-reserved-leases)
- [DHCP Lease Management](#dhcp-lease-management)
- [DHCP Client (WAN Interface)](#dhcp-client-wan-interface)
- [DHCP Relay](#dhcp-relay)
- [WinBox / WebFig Configuration](#winbox--webfig-configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

The DHCP server in RouterOS automatically assigns IP addresses, subnet masks, default gateways, DNS servers, and other options to clients on a network segment. Key components are:

| Component | Description |
|-----------|-------------|
| **IP Pool** | Range of IP addresses available for assignment |
| **DHCP Network** | Subnet definition with gateway, DNS, NTP, etc. |
| **DHCP Server** | Server instance bound to a specific interface |
| **DHCP Lease** | Record of an assigned IP address and its client |

---

## DHCP Server Quick Setup (Wizard)

RouterOS includes an interactive wizard that creates all DHCP components in one step:

```bash
/ip dhcp-server setup
```

The wizard will prompt for:
1. **DHCP server interface** (e.g., `bridge1`)
2. **DHCP address space** (e.g., `192.168.1.0/24`)
3. **Gateway** (e.g., `192.168.1.1`)
4. **Pool range** (e.g., `192.168.1.100-192.168.1.200`)
5. **DNS servers** (e.g., `192.168.1.1`)
6. **Lease time** (e.g., `10m` for testing, `1d` for production)

---

## Manual DHCP Server Setup

### Step 1: Create an IP Pool

An IP pool defines the range of addresses the server can hand out:

```bash
/ip pool add \
    name=pool-lan \
    ranges=192.168.1.100-192.168.1.200 \
    comment="LAN DHCP pool"
```

> Leave addresses outside the pool range for static assignments (e.g., `192.168.1.1–192.168.1.99` and `192.168.1.201–192.168.1.254`).

### Step 2: Create the DHCP Network

The network entry defines the subnet parameters sent to clients:

```bash
/ip dhcp-server network add \
    address=192.168.1.0/24 \
    gateway=192.168.1.1 \
    dns-server=192.168.1.1 \
    ntp-server=192.168.1.1 \
    domain=lan \
    comment="LAN network"
```

| Parameter | Description |
|-----------|-------------|
| `address` | Network address in CIDR notation |
| `gateway` | Default gateway sent to clients |
| `dns-server` | Comma-separated DNS servers |
| `ntp-server` | Optional NTP server |
| `domain` | DNS domain suffix for hostname search |
| `lease-time` | Override lease time for this network (optional) |

### Step 3: Create the DHCP Server Instance

Bind the DHCP server to an interface and point it to the pool:

```bash
/ip dhcp-server add \
    name=dhcp-lan \
    interface=bridge1 \
    address-pool=pool-lan \
    lease-time=1d \
    disabled=no \
    comment="LAN DHCP server"
```

---

## Multi-VLAN DHCP Server Example

Configure separate DHCP servers for three VLANs (assumes VLAN interfaces `vlan10`, `vlan20`, `vlan30` are already created — see [VLANs guide](vlans.md)):

```bash
# IP Pools
/ip pool add name=pool-vlan10 ranges=192.168.10.100-192.168.10.200
/ip pool add name=pool-vlan20 ranges=192.168.20.100-192.168.20.200
/ip pool add name=pool-vlan30 ranges=192.168.30.100-192.168.30.240

# DHCP Networks
/ip dhcp-server network add \
    address=192.168.10.0/24 \
    gateway=192.168.10.1 \
    dns-server=192.168.10.1 \
    domain=mgmt.lan \
    comment="VLAN10 Management"

/ip dhcp-server network add \
    address=192.168.20.0/24 \
    gateway=192.168.20.1 \
    dns-server=192.168.20.1 \
    domain=servers.lan \
    comment="VLAN20 Servers"

/ip dhcp-server network add \
    address=192.168.30.0/24 \
    gateway=192.168.30.1 \
    dns-server=192.168.30.1 \
    domain=clients.lan \
    comment="VLAN30 Clients"

# DHCP Server Instances
/ip dhcp-server add name=dhcp-vlan10 interface=vlan10 address-pool=pool-vlan10 lease-time=1d disabled=no
/ip dhcp-server add name=dhcp-vlan20 interface=vlan20 address-pool=pool-vlan20 lease-time=12h disabled=no
/ip dhcp-server add name=dhcp-vlan30 interface=vlan30 address-pool=pool-vlan30 lease-time=8h disabled=no
```

---

## DHCP Options

### Common Options

DHCP options are set per network in the `/ip dhcp-server network` entry.

| Parameter | DHCP Option | Description |
|-----------|-------------|-------------|
| `gateway` | Option 3 | Default gateway |
| `dns-server` | Option 6 | DNS servers (up to 3) |
| `domain` | Option 15 | DNS domain name |
| `ntp-server` | Option 42 | NTP servers |
| `wins-server` | Option 44 | WINS server (for Windows environments) |
| `boot-file-name` | Option 67 | PXE boot file |
| `next-server` | Option 66 | TFTP server for PXE boot |

### Custom DHCP Options

For non-standard options (e.g., VOIP provisioning, option 43, option 120):

```bash
# Define option 120 (SIP server for VoIP phones)
/ip dhcp-server option add \
    name=sip-server \
    code=120 \
    value="'sip.example.com'" \
    comment="SIP server for VoIP"

# Apply option to a specific network
/ip dhcp-server network set [find address=192.168.20.0/24] dhcp-option=sip-server

# Define option set (multiple options bundled)
/ip dhcp-server option sets add \
    name=voip-options \
    options=sip-server
```

TFTP/PXE boot configuration:

```bash
/ip dhcp-server network set [find address=192.168.1.0/24] \
    next-server=192.168.1.10 \
    boot-file-name=pxelinux.0
```

---

## Static (Reserved) Leases

Reserve a specific IP address for a client based on its MAC address:

```bash
# Reserve IP for a server
/ip dhcp-server lease add \
    address=192.168.1.50 \
    mac-address=AA:BB:CC:DD:EE:01 \
    server=dhcp-lan \
    comment="File server - static lease"

# Reserve IP for a printer
/ip dhcp-server lease add \
    address=192.168.1.60 \
    mac-address=AA:BB:CC:DD:EE:02 \
    server=dhcp-lan \
    comment="Office printer - static lease"
```

Convert an existing dynamic lease to a static one:

```bash
# Find the lease and make it static
/ip dhcp-server lease make-static [find host-name="myserver"]
```

---

## DHCP Lease Management

```bash
# View all current leases
/ip dhcp-server lease print

# View leases for a specific server
/ip dhcp-server lease print where server=dhcp-lan

# View active (bound) leases only
/ip dhcp-server lease print where status=bound

# Find lease by IP
/ip dhcp-server lease print where address=192.168.1.105

# Remove a specific lease (force renewal)
/ip dhcp-server lease remove [find address=192.168.1.105]

# Remove all expired leases
/ip dhcp-server lease remove [find status=expired]
```

---

## DHCP Client (WAN Interface)

Configure the router to obtain its WAN IP address from the ISP via DHCP:

```bash
/ip dhcp-client add \
    interface=ether1 \
    add-default-route=yes \
    default-route-distance=1 \
    use-peer-dns=yes \
    use-peer-ntp=yes \
    disabled=no \
    comment="WAN DHCP client"
```

| Parameter | Description |
|-----------|-------------|
| `add-default-route` | Automatically add a default route using the gateway from DHCP |
| `default-route-distance` | Distance for the added default route |
| `use-peer-dns` | Use DNS servers provided by the DHCP server |
| `use-peer-ntp` | Use NTP servers provided by the DHCP server |

View DHCP client status:

```bash
/ip dhcp-client print detail
```

Renew a DHCP lease:

```bash
/ip dhcp-client renew [find interface=ether1]
```

---

## DHCP Relay

When the DHCP server is on a separate device (e.g., a centralized server at `10.0.0.53`), use DHCP relay to forward requests from each VLAN:

```bash
/ip dhcp-relay add \
    name=relay-vlan10 \
    interface=vlan10 \
    dhcp-server=10.0.0.53 \
    local-address=192.168.10.1 \
    disabled=no \
    comment="Relay for VLAN10"

/ip dhcp-relay add \
    name=relay-vlan20 \
    interface=vlan20 \
    dhcp-server=10.0.0.53 \
    local-address=192.168.20.1 \
    disabled=no \
    comment="Relay for VLAN20"
```

> **Note:** When using relay, the DHCP server must have network entries for each relayed subnet.

---

## WinBox / WebFig Configuration

### DHCP Server

1. **IP** → **DHCP Server** → **DHCP** tab → Click **+**
2. Set **Name**, **Interface**, **Lease Time**, and **Address Pool**
3. Click **OK**

### DHCP Network

1. **IP** → **DHCP Server** → **Networks** tab → Click **+**
2. Fill in **Address**, **Gateway**, **DNS Server**
3. Click **OK**

### Static Leases

1. **IP** → **DHCP Server** → **Leases** tab → Click **+**
2. Fill in **Address** and **MAC Address**
3. Click **OK**

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Clients not getting an IP | DHCP server disabled or wrong interface | Check server is enabled and bound to the correct interface |
| IP assigned but no internet | Missing or wrong default gateway in network entry | Verify `gateway=` in `/ip dhcp-server network` |
| Static lease not applying | MAC address mismatch | Double-check MAC with `/ip dhcp-server lease print` |
| Pool exhausted | Pool range too small | Reduce lease time or expand the pool range |
| Duplicate IPs on the network | Static IP set within pool range | Set static IPs outside the pool range |

```bash
# Check DHCP server status
/ip dhcp-server print detail

# View active leases and pool utilization
/ip dhcp-server lease print count-only

# Check for errors in DHCP log
/log print where topics~"dhcp"
```
