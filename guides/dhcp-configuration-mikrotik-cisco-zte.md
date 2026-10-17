# DHCP Configuration Field Guide

<div align="center">

![DHCP](https://img.shields.io/badge/NETWORKING-DHCP%20FIELD%20GUIDE-06B6D4?style=for-the-badge&labelColor=0B1220)
![MikroTik](https://img.shields.io/badge/MIKROTIK-RouterOS-293239?style=for-the-badge&logo=mikrotik&logoColor=white)
![Cisco](https://img.shields.io/badge/CISCO-IOS%20%2F%20IOS%20XE-049FD9?style=for-the-badge&logo=cisco&logoColor=white)
![ZTE](https://img.shields.io/badge/ZTE-HOME%20GATEWAYS-0072CE?style=for-the-badge)

**Plan scopes · assign leases · reserve addresses · relay requests · troubleshoot clients**

</div>

> A practical English-language learning guide for DHCP on MikroTik, Cisco, ZTE, Huawei, and D-Link equipment. CLI examples use a lab network; adapt interface names, subnets, and addresses before applying them.

---

## Contents

1. [DHCP in one minute](#1-dhcp-in-one-minute)
2. [Plan the address scope](#2-plan-the-address-scope)
3. [MikroTik RouterOS](#3-mikrotik-routeros)
4. [Cisco IOS and IOS XE](#4-cisco-ios-and-ios-xe)
5. [ZTE, Huawei, and D-Link](#5-zte-huawei-and-d-link)
6. [DHCP client and relay](#6-dhcp-client-and-relay)
7. [Verification and troubleshooting](#7-verification-and-troubleshooting)
8. [Security and operational checklist](#8-security-and-operational-checklist)
9. [Official references](#9-official-references)

## 1. DHCP in one minute

DHCP automatically provides a client with network settings such as an IP address, subnet mask, default gateway, DNS servers, and lease time. A typical IPv4 exchange is called **DORA**:

| Stage | Message | What happens |
|:--|:--|:--|
| **D**iscover | Client → broadcast | Client searches for a DHCP server. |
| **O**ffer | Server → client | Server offers an address and options. |
| **R**equest | Client → broadcast/server | Client requests the offered configuration. |
| **A**cknowledgment | Server → client | Server confirms the lease. |

A DHCP server normally serves clients on its local broadcast domain. To serve a different subnet, configure a **DHCP relay** on the Layer 3 gateway, and configure a matching scope on the central server.

## 2. Plan the address scope

This guide uses a small lab subnet:

| Setting | Example |
|:--|:--|
| LAN subnet | `192.168.10.0/24` |
| Gateway | `192.168.10.1` |
| Dynamic client pool | `192.168.10.100–192.168.10.199` |
| Reserved/static devices | `192.168.10.2–192.168.10.99` |
| DNS servers | `1.1.1.1`, `1.0.0.1` |
| Example lease time | 1 day |

Before configuring DHCP:

1. Confirm the subnet and gateway on the router interface or VLAN interface.
2. Select a pool that does not overlap with the gateway, statically addressed devices, or another DHCP server's pool.
3. Identify the device that will be the **only** DHCP server for this subnet.
4. Decide what clients should receive: gateway, DNS, domain name, NTP, and lease duration.
5. Keep a configuration backup and access to the device if the change interrupts management.

> Never run a second DHCP server on the same LAN by accident. Rogue or overlapping DHCP servers can give clients the wrong gateway or DNS.

## 3. MikroTik RouterOS

The following is a RouterOS v7 lab example. It assumes that a bridge called `bridge-lan` already exists and has `192.168.10.1/24` assigned. Adjust the bridge and subnet to match your device.

### Create a pool, scope, and DHCP server

```routeros
/ip pool add name=pool-lan ranges=192.168.10.100-192.168.10.199
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=1.1.1.1,1.0.0.1
/ip dhcp-server add name=dhcp-lan interface=bridge-lan address-pool=pool-lan lease-time=1d disabled=no
```

If the router itself is the LAN DNS cache, use `dns-server=192.168.10.1` in the DHCP network entry and configure DNS on the router separately. See the [DNS Configuration Field Guide](dns-configuration-mikrotik-cisco-zte.md).

### Change DHCP options or lease duration

If the network entry already exists, update it instead of adding a duplicate:

```routeros
/ip dhcp-server network print detail
/ip dhcp-server network set [find where address="192.168.10.0/24"] gateway=192.168.10.1 dns-server=1.1.1.1,1.0.0.1
/ip dhcp-server set [find where name="dhcp-lan"] lease-time=1d
```

A shorter lease can help in guest networks where devices frequently come and go. A longer lease can reduce renewal traffic on stable networks. Choose a value appropriate to the network.

### Add a reserved address (static lease)

A static DHCP lease ties an address to a client's MAC address. Keep the reserved address outside the dynamic pool:

```routeros
/ip dhcp-server lease add server=dhcp-lan mac-address=AA:BB:CC:DD:EE:FF address=192.168.10.20 comment="Printer"
```

Use the client's actual MAC address and an unused address from the subnet. A reservation is different from configuring a static IP directly on the client.

### Verify MikroTik configuration

```routeros
/ip pool print
/ip dhcp-server print detail
/ip dhcp-server network print detail
/ip dhcp-server lease print detail
```

## 4. Cisco IOS and IOS XE

This example configures a Cisco IOS/IOS XE device as the DHCP server for the lab subnet. It assumes the gateway interface already uses `192.168.10.1/24`.

### Configure a DHCP pool

```text
enable
configure terminal
ip dhcp excluded-address 192.168.10.1 192.168.10.99
ip dhcp pool LAN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 1.1.1.1 1.0.0.1
 lease 1
end
copy running-config startup-config
```

The excluded range prevents the server from offering addresses reserved for the gateway and static devices. The interface address is automatically excluded on supported IOS/IOS XE DHCP server implementations, but the explicit range documents the pool plan and avoids allocating other reserved addresses.

### Create a DHCP reservation for one client

For IOS/IOS XE platforms that support manual DHCP bindings, use a host pool and the client's hardware address. Confirm syntax on the target platform and release before deployment:

```text
configure terminal
ip dhcp pool PRINTER-RESERVATION
 host 192.168.10.20 255.255.255.0
 hardware-address aabb.ccdd.eeff ieee802
 default-router 192.168.10.1
 dns-server 1.1.1.1 1.0.0.1
end
copy running-config startup-config
```

Some Cisco platforms and software releases use different reservation workflows. Do not add a manual binding to a conflicting dynamic pool; verify the address is excluded or otherwise protected from duplicate assignment.

### Verify Cisco DHCP

```text
show running-config | section ip dhcp
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
```

## 5. ZTE, Huawei, and D-Link

Home gateway interfaces vary by **model, firmware, and ISP customization**. The exact screens and supported fields can differ, so these steps describe the common workflow rather than a universal menu path.

### Configure the DHCP server in the web interface

1. Connect to the router and open its gateway address in a browser, often `192.168.1.1` or `192.168.0.1`.
2. Sign in with your administrator credentials.
3. Find **Local Network**, **LAN**, or **Home Network**.
4. Open **DHCP Server** or **DHCPv4 Server** and enable it only if this device should provide DHCP for the subnet.
5. Set the starting and ending address range. Keep the gateway, infrastructure, and statically assigned addresses outside the dynamic pool.
6. Set the subnet mask, default gateway, DNS servers, and lease duration if the interface exposes those fields.
7. Save the configuration, renew a client lease, and verify the client received the intended values.

### Reserve an address

Look for **Address Reservation**, **Static DHCP**, **DHCP Binding**, or **IP/MAC Binding**. Select the client's MAC address and assign an unused address outside the dynamic pool where supported.

### When the router is an access point

If a MikroTik or Cisco device already provides DHCP, disable the DHCP server on the ZTE/Huawei/D-Link access point or secondary router. Connect it according to the network design and avoid creating a second scope on the same LAN.

**For exact screenshots and matching menu names**, the router model and firmware version are required. Do not share administrator passwords.

## 6. DHCP client and relay

### MikroTik as a DHCP client on a WAN interface

Use this when the ISP or upstream router assigns the MikroTik WAN address dynamically. First check whether a DHCP client already exists on the interface.

```routeros
/ip dhcp-client add interface=ether1 add-default-route=yes use-peer-dns=yes disabled=no
/ip dhcp-client print detail
```

Replace `ether1` with the actual WAN interface. Do not add a second client if one is already configured.

### Cisco interface as a DHCP client

On a routed interface that should obtain its address from an upstream DHCP server:

```text
configure terminal
interface GigabitEthernet0/0
 ip address dhcp
 no shutdown
end
```

Use the interface name supported by the device. Do not use this on an interface that must keep a static address.

### Cisco DHCP relay with `ip helper-address`

When the DHCP server is on another subnet, configure the relay on the client VLAN's Layer 3 interface. Example: clients are on VLAN 10, and the central DHCP server is `10.0.0.10`.

```text
configure terminal
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 10.0.0.10
 no shutdown
end
```

The server needs a scope for `192.168.10.0/24`, and routing must work in both directions. If the switch/router uses a different gateway SVI or relay syntax, adapt the example to the actual platform.

### MikroTik DHCP relay

Use a relay when the DHCP server is on another subnet. Example: clients are on `192.168.10.0/24`, the relay's client-facing interface is `bridge-lan`, and the central server is reachable at `10.0.0.10`.

```routeros
/ip dhcp-relay add name=relay-lan interface=bridge-lan dhcp-server=10.0.0.10 local-address=192.168.10.1 disabled=no
```

Configure the central server with a scope for the client subnet and the correct gateway. Ensure routing and firewall rules allow DHCP relay traffic between the relay and server.

## 7. Verification and troubleshooting

Check these items in order:

1. **Client link and VLAN:** confirm the device is connected to the intended network/VLAN.
2. **Addressing:** verify the subnet, gateway, and pool are correct and do not overlap.
3. **One server per scope:** check for an unintended second DHCP server.
4. **Lease state:** verify the client has a current lease and the server has available addresses.
5. **Options:** check that the client received the expected gateway, DNS, and lease duration.
6. **Relay path:** for routed clients, verify `ip helper-address` or DHCP relay, routing, server scope, and firewall policy.
7. **Renew the lease** after changing DHCP settings.

On Windows, run:

```powershell
ipconfig /all
ipconfig /release
ipconfig /renew
```

Use `ipconfig /release` and `/renew` only on a client machine, not on the router or a remote server session that depends on that connection.

Useful device checks:

| Platform | Commands |
|:--|:--|
| MikroTik | `/ip dhcp-server print detail`, `/ip dhcp-server lease print detail`, `/ip dhcp-server network print detail` |
| Cisco | `show ip dhcp pool`, `show ip dhcp binding`, `show ip dhcp conflict`, `show running-config | section ip dhcp` |

## 8. Security and operational checklist

- Use one authoritative DHCP server per broadcast domain unless you intentionally design redundancy or failover.
- Exclude gateway, network infrastructure, servers, printers, and other static addresses from dynamic pools.
- Use reservations for centrally managed devices when practical; document address, MAC, device, and purpose.
- Restrict administrative access to routers and DHCP servers.
- Use DHCP snooping on supported switches when the network design calls for it, and carefully mark trusted uplinks.
- Back up the configuration and keep a management path before changing a production scope or relay.
- Do not expose DHCP server services to untrusted networks.
- DHCPv6 and IPv6 Router Advertisements are separate from this IPv4 guide; use the platform's IPv6 documentation for those designs.

## 9. Official references

- [MikroTik RouterOS — DHCP](https://help.mikrotik.com/docs/display/ROS/DHCP)
- [MikroTik RouterOS — IP Pools](https://help.mikrotik.com/docs/display/ROS/IP+Pools)
- [MikroTik RouterOS — First Time Configuration](https://help.mikrotik.com/docs/display/ROS/First+Time+Configuration)
- [Cisco IOS XE 17 — DHCP Server Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_config-dhcp-server-xe.html)
- [Cisco IOS XE — DHCP Relay Agent](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_dhcp/configuration/xe-16-11/dhcp-xe-16-11-book/dhcp-relay-agent-xe.html)
- [ZTE Support](https://www.zte.com.cn/global/support)
- [Huawei Support](https://consumer.huawei.com/en/support/)
- [D-Link Support](https://support.dlink.com/)

---

<div align="center">

**Plan the scope. Assign the lease. Verify the client.**

</div>
