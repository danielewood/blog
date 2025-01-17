---
author: ["Daniel Wood"]
title: "Resetting and Upgrading the APC AP9606 NMC"
date: "2019-04-05"
description: "A guide to resetting, upgrading, and configuring the APC AP9606 Network Management Card, including firmware updates and SNMP time configuration"
summary: "Learn how to work with the legacy APC AP9606 NMC, including IP reset procedures, firmware upgrades, and SNMP configuration. Perfect for setting up isolated VLAN monitoring of Smart-UPS systems."
tags: ["apc", "ups", "snmp", "networking", "firmware", "ap9606", "nmc"]
categories: ["hardware", "networking", "tutorials"]
series: ["ups-management"]
ShowToc: true
TocOpen: true
aliases: 
  - /2019/04/resetting-and-upgrading-apc-ap9606.html
---

The APC AP9606 is a very old NMC for the Smart-UPS line. They go for under $10 each on ebay and are great for use on an isolated VLAN for SNMP alerts and actions. Their network interface is only 10BaseT, but that is a non-issue for the purpose they serve and they work without issue on most gigabit switches.

## General Notes
- The default username/password is apc/apc.
- The AP9606 does not support DHCP, it supports only Static IP addresses or BOOTP.
- Setting the time on the AP9606 is broken on both the WebUI and on Telnet. [You can set it through SNMP](https://github.com/danielewood/misc/issues/3).
- Unauthenticated SMTP Mail alerts work with GSuite SMTP Relay, as long as you have whitelisted your WAN IP.

## Discover/Reset IP Address
1. Connect to the same layer 2 network as a PC.
1. Use Wireshark and filter for: `eth.src[0:3]==00:c0:b7`
1. Reset the AP9606 (hold reset button for 2-3 seconds)
    - After about 30 seconds, you will see the AP9606 doing ARP lookups.
1. Put your computer in the same IP range and login (apc/apc).
1. Go to `System | Tools | Action | Reset to Defaults except TCP/IP`
    - This is to clear any custom settings someone else may have had.
1. After the card reboots (~30 seconds), log back in and set your own IP settings.
1. After you set your desired IP address, logout to initiate a reboot and IP change.

## Upgrade firmware
1. Download latest available firmwares ([FTP Search Engines](https://www.searchftps.net/) are useful here).
    1. ```
       Description : Web/SNMP Management Card AOS
       -----------------------------------------------------------------------
       Name        : aos326b.bin       Type        : APC OS
       Version     : v3.2.6.b          Sector      : 11
       Date        : 09/16/2003        Time        : 07:45:28
       CRC16       : 679F
       MD5         : 381f98dea7694b51b125d61916811d1f
       ```
    1. ```
       Description : Smart-UPS & Matrix-UPS APP
       -----------------------------------------------------------------------
       Name        : sumx326.bin       Type        : StatApp
       Version     : v3.2.6            Sector      : 4
       Date        : 08/14/2002        Time        : 17:01:44
       CRC16       : 8545
       MD5         : a7faf768e0e2437602c939dfcfb5a15b
       ```
1. Use curl to ftp upload the `Web/SNMP Management Card AOS`:
    - `curl -T aos326b.bin --ftp-port - ftp://IP_ADDRESS_OF_AP9606 --user apc:apc`
1. Wait 30 seconds for card to reboot.
1. Use curl to ftp upload the `Smart-UPS & Matrix-UPS APP`:
    - `curl -T sumx326.bin --ftp-port - ftp://IP_ADDRESS_OF_AP9606 --user apc:apc`
1. Wait 30 seconds for card to reboot.
1. Done, you are now on the newest available firmware for the AP9606


## Set Time and Date through SNMP
Note: You must not be logged into the WebUI when running this.

Run the following bash, replacing the IP addresses with your own:

```bash
#!/bin/bash
APC_IPs='172.30.1.170
172.30.1.171
172.30.1.172'

while IFS= read -r line ; do
    DATE=$(date +%m/%d/%Y)
    snmpset -cprivate -v1 $line 1.3.6.1.4.1.318.2.1.6.1.0 s "$DATE" > /dev/null
    TIME=$(date +%H:%M:%S)
    snmpset -cprivate -v1 $line 1.3.6.1.4.1.318.2.1.6.2.0 s "$TIME" > /dev/null
done <<< "$APC_IPs"
```
