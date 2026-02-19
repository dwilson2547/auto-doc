# MikroTik Networking Guide

This directory contains a comprehensive set of guides for setting up and configuring networks using MikroTik equipment (RouterOS). Whether you are deploying a small office network or a more complex multi-site infrastructure, these documents cover the most common configuration tasks.

## Table of Contents

| Document | Description |
|----------|-------------|
| [Static Routing](static-routing.md) | Configure static routes, default gateways, and route management |
| [VLANs](vlans.md) | VLAN setup on MikroTik routers and switches |
| [DNS Configuration](dns-config.md) | DNS client, DNS cache/resolver, and static DNS entries |
| [DHCP Configuration](dhcp-config.md) | DHCP server and client configuration with address pools |
| [Spanning Tree](spanning-tree.md) | STP, RSTP, and MSTP to prevent Layer 2 loops |
| [Trunk & Access Ports](trunk-access-ports.md) | Configuring trunk and access ports on CRS/CSS switches |
| [QoS](qos.md) | Quality of Service with queues, Mangle, and traffic prioritization |
| [RIP & Routing Protocols](rip-routing-protocols.md) | RIP, OSPF, and BGP overview and configuration examples |

## Prerequisites

- MikroTik router or switch running **RouterOS v6.49+** or **RouterOS v7.x**
- Access to the device via **WinBox**, **WebFig**, or **SSH/Terminal**
- Basic understanding of IP networking concepts

## RouterOS CLI Conventions

All CLI examples in these guides use the MikroTik RouterOS terminal syntax. Key conventions:

```
/ip route add       # Add an IP route
/interface bridge   # Work with bridge interfaces
/tool export        # Export configuration
```

> **Tip:** Prefix any command with a `?` to see context-sensitive help in the RouterOS terminal, e.g. `/ip route add ?`

## RouterOS Versions

Some features differ between **RouterOS v6** and **RouterOS v7**. Where relevant, version-specific notes are provided in each document. RouterOS v7 introduced a revised bridge implementation (with hardware offload support) and updated VLAN handling that differs from v6.

## Additional Resources

- [MikroTik Documentation](https://help.mikrotik.com/docs/)
- [MikroTik Forum](https://forum.mikrotik.com/)
- [RouterOS Changelog](https://mikrotik.com/download/changelogs)
- [MikroTik Wiki](https://wiki.mikrotik.com/)
