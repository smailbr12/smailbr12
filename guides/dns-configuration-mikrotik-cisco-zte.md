# DNS Configuration Guide for Network Devices

<div align="center">

![DNS & Networking](https://img.shields.io/badge/NETWORKING-DNS%20CONFIGURATION-06B6D4?style=for-the-badge&labelColor=0B1220)
![MikroTik](https://img.shields.io/badge/MIKROTIK-RouterOS-293239?style=for-the-badge&logo=mikrotik&logoColor=white)
![Cisco](https://img.shields.io/badge/CISCO-IOS%20%2F%20IOS%20XE-049FD9?style=for-the-badge&logo=cisco&logoColor=white)
![ZTE](https://img.shields.io/badge/ZTE-HOME%20GATEWAYS-0072CE?style=for-the-badge)

**Configure DNS and understand the difference between a router's DNS settings and the DNS servers clients receive through DHCP.**

</div>

> Practical learning guide. Commands use a lab network as an example. Replace the sample addresses with values that match your network before applying them.

---

## Contents

1. [What are we changing?](#1-what-are-we-changing)
2. [Lab plan](#2-lab-plan)
3. [MikroTik RouterOS](#3-mikrotik-routeros)
4. [Cisco IOS and IOS XE](#4-cisco-ios-and-ios-xe)
5. [ZTE, Huawei, and D-Link](#5-zte-huawei-and-d-link)
6. [Verification and troubleshooting](#6-verification-and-troubleshooting)
7. [DNS security](#7-dns-security)
8. [Official references](#8-official-references)

## 1. What are we changing?

DNS translates names such as `example.com` into IP addresses. When changing DNS on a router, keep these separate settings in mind:

| Setting | Purpose |
|:--|:--|
| **DNS upstream / WAN** | DNS servers the router itself queries, or forwards client requests to. |
| **DNS advertised over DHCP / LAN** | DNS server addresses the router gives network clients when they receive an IP address. |
| **Router DNS cache** | Lets the router receive DNS requests from LAN clients and reuse cached responses. |

**Example:** The router can use public DNS servers upstream while DHCP gives clients the router's address, `192.168.10.1`. In that design, the router should accept DNS requests from the LAN only.

## 2. Lab plan

The examples use these values:

| Item | Lab value |
|:--|:--|
| LAN network | `192.168.10.0/24` |
| Gateway/router address | `192.168.10.1` |
| Primary and secondary DNS | `1.1.1.1` and `1.0.0.1` |

These addresses are examples from Cloudflare Public DNS. You can use your ISP's DNS or an internal resolver instead. See [Cloudflare's official resolver addresses](https://developers.cloudflare.com/1.1.1.1/ip-addresses/).

## 3. MikroTik RouterOS

The commands below are examples for RouterOS v7. Before making changes, export a backup of your configuration and verify your device's DHCP network name and subnet.

### Option A — Use the router as a DNS cache for the LAN

This sets upstream DNS servers, enables DNS service for clients, and advertises the router's address through DHCP:

```routeros
/ip dns set servers=1.1.1.1,1.0.0.1 allow-remote-requests=yes
/ip dhcp-server network set [find where address="192.168.10.0/24"] dns-server=192.168.10.1 gateway=192.168.10.1
```

If the DHCP network entry does not exist yet, add it after confirming the rest of your configuration uses the same addresses:

```routeros
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
```

### Option B — Give clients public DNS directly

Use this option if you do not want MikroTik to act as the clients' DNS cache:

```routeros
/ip dhcp-server network set [find where address="192.168.10.0/24"] dns-server=1.1.1.1,1.0.0.1
```

With this option, clients connect directly to the DNS servers, so you do not need to enable `allow-remote-requests` for them.

### Verify on MikroTik

```routeros
/ip dns print
/ip dhcp-server network print detail
/ip dns cache print
```

After renewing DHCP on a client, verify that its DNS list shows `192.168.10.1` for Option A, or `1.1.1.1` and `1.0.0.1` for Option B.

## 4. Cisco IOS and IOS XE

The `ip name-server` command configures name servers used by the Cisco device for name resolution. The `dns-server` command inside a DHCP pool sends DNS addresses to clients.

### Configure router DNS and advertise DNS to DHCP clients

Example for a Cisco device providing DHCP on the lab network:

```text
enable
configure terminal
ip name-server 1.1.1.1 1.0.0.1
ip dhcp pool LAN
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 1.1.1.1 1.0.0.1
end
copy running-config startup-config
```

> If another device provides DHCP, do not create a second DHCP pool. Update the DNS option on the active DHCP server instead. If your pool has a different name, use it instead of `LAN`.

### Verification commands

```text
show running-config | include ip name-server
show running-config | section ip dhcp pool
show ip dhcp binding
ping example.com
```

If `ping example.com` fails while a ping to an IP address succeeds, investigate DNS. The result also depends on ICMP policy and internet connectivity.

## 5. ZTE, Huawei, and D-Link

Management menus and field names vary by **model, firmware version, and ISP**. Home CPE devices do not have one CLI command set that can be guaranteed across all ZTE, Huawei, or D-Link models. Avoid copying Telnet/SSH commands from a different model.

### General web interface steps

1. Connect to the router and open its gateway address in a browser, such as `192.168.1.1` or `192.168.0.1`, depending on the device.
2. Sign in with your own administrator credentials.
3. To change the DNS servers the router uses for internet access, look under **Internet / WAN / Broadband**, open the active internet connection, and find **DNS Mode** or **Primary DNS / Secondary DNS**.
4. To change the DNS servers given to LAN devices, look under **Local Network / LAN / DHCP Server** for **DNS Server** or **DHCP DNS**.
5. Enter the desired addresses, save the change, then renew DHCP on a client device.
6. Verify which DNS address the client received before making any additional changes.

On some ZTE interfaces, the settings may appear under **Network → LAN → DHCP Server** or **Internet → WAN**. These are common labels, not guaranteed menu paths. On Huawei and D-Link devices, look for the same functions even if the menu names differ.

**For exact steps and matching screenshots**, the device model and firmware version are required, such as `ZTE ZXHN H168N` or the model printed on the label. Do not share the administrator password.

## 6. Verification and troubleshooting

Run these checks in order:

1. **Check the client's IP address, gateway, and DNS.** On Windows:
   ```powershell
   ipconfig /all
   ```
2. **Renew DHCP** after changing the DNS option. Existing clients may keep the old setting until their lease renews or they reconnect.
3. **Test access to an IP address**, then test a domain name. If the IP works but the name does not, focus on DNS.
4. **Check the router's connection to its DNS server** and verify that firewall rules or ISP policies are not blocking UDP/TCP port 53.
5. **Clear the Windows DNS cache** if needed:
   ```powershell
   ipconfig /flushdns
   ```
6. **On Cisco**, review `show ip dhcp binding` and confirm the client received the expected lease, then inspect the DHCP pool.
7. **On MikroTik**, review DNS and DHCP network settings, then check firewall rules that control DNS requests.

## 7. DNS security

- Do not leave a DNS cache open to requests from any internet source. When `allow-remote-requests=yes` is enabled on MikroTik, allow DNS requests only from trusted LAN clients and block them from the WAN.
- Traditional DNS over port 53 is not encrypted by default. Follow your network policy, and do not change DNS on an organization network without the administrator's approval.
- Use primary and secondary DNS addresses that are trusted and reachable from the router.
- Record the previous settings before changing them. Change one setting at a time so it is easier to recover if name resolution stops working.
- Before applying DHCP commands to a live network, verify the subnet, gateway, pool name, and DHCP network.

## 8. Official references

- [MikroTik RouterOS — DNS](https://help.mikrotik.com/docs/display/ROS/DNS)
- [MikroTik RouterOS — DHCP](https://help.mikrotik.com/docs/display/ROS/DHCP)
- [Cisco IOS XE 17 — DHCP Server Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_config-dhcp-server-xe.html)
- [Cisco IOS IP Addressing Services — Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr/command/ipaddr-i3.html)
- [Cloudflare 1.1.1.1 — Resolver addresses](https://developers.cloudflare.com/1.1.1.1/ip-addresses/)
- [ZTE Support](https://www.zte.com.cn/global/support)
- [Huawei Support](https://consumer.huawei.com/en/support/)
- [D-Link Support](https://support.dlink.com/)

---

<div align="center">

**Reliable DNS configuration starts with understanding the difference between the DNS server the router uses and the DNS server the client receives.**

</div>
