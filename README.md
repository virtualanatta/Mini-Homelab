# Mini-Homelab
# 1. Project Overview

This document outlines the end-to-end deployment of a security gateway
using the GL-MT300N-V2 (Mango) router. The objective is to centralize
network security, ad-blocking, and traffic management into a portable,
low-cost device.

------------------------------------------------------------------------

# 2. Prerequisites and Hardware

-   Device: GL-iNet GL-MT300N-V2.
-   Storage: USB 2.0/3.0 Flash Drive (16GB minimum).
-   Power: 5V/2A Micro-USB adapter.
-   Connectivity: Ethernet cable for initial setup.

------------------------------------------------------------------------

# 3. Initial Configuration and Environment Setup

## 3.1 First Access

-   Connect the router to power and plug your PC into the LAN port.
-   Navigate to:

``` bash
```
    http://192.168.8.1

-   Complete the initial wizard:
    -   Language
    -   Admin Password

------------------------------------------------------------------------

## 3.2 External Storage Preparation (Extroot)

To install advanced packages, the internal 16MB flash must be expanded
via the USB drive.

### Format the USB

Access the router via SSH and identify the drive:

``` bash
lsblk
```

Create a Linux partition (ext4) on the USB:

``` bash
mkfs.ext4 /dev/sda1
```

### Mounting

Navigate to:

    LuCI > System > Mount Points

Configure:

    /dev/sda1

as the new external overlay (Extroot).

Reboot the device.

Verify expansion with:

``` bash
df -h
```

------------------------------------------------------------------------

# 4. Software Installation and Services

## 4.1 Dependency Installation

Update the package lists and install the core components:

``` bash
opkg update
opkg install luci-app-statistics luci-app-sqm adguardhome
```

------------------------------------------------------------------------

## 4.2 Traffic Management (SQM)

-   Navigate to:

```{=html}
<!-- -->
```
    LuCI > Network > SQM QoS

-   Enable on the `eth0 (WAN)` interface.
-   Set download/upload speeds to **90% of your actual bandwidth** to
    prevent bufferbloat.

------------------------------------------------------------------------

# 5. AdGuard Home Deployment

## 5.1 Service Initialization

Access the AdGuard setup wizard:

    http://192.168.8.1:3000

Configuration:

-   Web Interface Port: `3000`
-   DNS Server Port: `3001` (to avoid conflicts with the system's
    dnsmasq)

------------------------------------------------------------------------

## 5.2 AdGuard Configuration

### Upstream DNS

    https://dns.cloudflare.com/dns-query
    https://dns.google/dns-query
    tls://dns.quad9.net

for encrypted (DoT) lookups.

### Filters

Enable:

-   AdGuard DNS Filter
-   EasyList

### Client Settings

Verify that the **Total Queries** counter increases during browsing.

------------------------------------------------------------------------

# 6. Networking and Firewall Persistence

## 6.1 DNS Interception

Since the router defaults to port 53, we must force traffic into AdGuard
(port 3001) via iptables.

Go to:

    GL.iNet Admin Panel > Network > DNS

Enable:

    Override DNS Settings for All Clients

### Manual Persistence

Edit:

    /etc/firewall.user

or add rules via LuCI Custom Rules.

``` bash
iptables -t nat -A PREROUTING -i br-lan -p udp --dport 53 -j DNAT --to 192.168.8.1:3001
iptables -t nat -A PREROUTING -i br-lan -p tcp --dport 53 -j DNAT --to 192.168.8.1:3001
```

------------------------------------------------------------------------

## 6.2 LED and Indicator Management

If the system reassigns LED paths after Extroot.

### Check paths

``` bash
ls /sys/class/leds/
```

### Force Trigger

``` bash
echo mmc0 > /sys/class/leds/red:wlan/trigger
```

to link LED activity to system/USB load.

------------------------------------------------------------------------

# 7. Diagnostics and Maintenance

## 7.1 Validation Commands

### Process Status

``` bash
netstat -tulpn | grep AdGuard
```

Verify ports `3000` and `3001`.

### Storage Integrity

``` bash
df -h
```

Verify `/overlay` is on `/dev/sda1`.

### Filtering Test

Run from a client machine:

``` bash
nslookup google.com 192.168.8.1
```

------------------------------------------------------------------------

## 7.2 Known Issues

### Cache Latency

Browsers may store ads. Test changes in **Incognito Mode**.

### Mount Latency

If AdGuard fails to start after a reboot, the USB mount might be
delayed.

``` bash
/etc/init.d/adguardhome restart
```
