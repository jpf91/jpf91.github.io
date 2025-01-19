---
title: Quick TFTP Recovery Setup
author:
  - Johannes Pfau
description: How to set up a TFTP server using podman
tags:
  - Fedora
  - IT
categories: Network
draft: false
---

Sometimes you need to recover a bricked device via TFTP, but you probably don't have a TFTP server available. So here's how to quickly set one on your local PC.

<!--more-->

I recently had to recover a broken GXP1625 IP-Phone, but the [recovery guide](https://tanshiyiing.com/phones-are-stuck-at-recovery-incomplete/) assumed a Windows installation.
Here's how to quickly replicate that setup on linux, without having to make any permanent changes to your system.

## Disable Firewall

Let's disable the firewall for now so that it does not interfere:
```bash
sudo systemctl stop firewalld
```

## Run DHCP Server

Open your network configuration UI and assign the static IP address `192.168.7.1` to the network interface you want to use for debugging.
If your bricked device supports recovering from a specific static address (usually home network routers), use that one of course.

Now let's run a DHCP server:
```bash
# dnsmasq is usually installed. If not, use podman like for the tftp server
sudo dnsmasq --interface=enp1s0 --port 0 --no-daemon --dhcp-range=192.168.7.50,192.168.7.100,12h --dhcp-option=66,192.168.7.1
```

In this command, we disable the DNS server (`--port 0`), start a DHCP server on interface `enp1s0`.
We also set DHCP option 66 to tell our IP-Phone there's a TFTP server for provisioning at `192.168.7.1`.


{{< box info >}}
  If dnsmasq complains that the address is already in use, you may have another dnsmasq running already.
  In that case, kill it:
  ```bash
  sudo pkill dnsmasq
  ```
{{< /box >}}

## Run TFTP Server

To finally run the TFTP server:
```bash
sudo podman run --rm --net host -v .:/var/tftpboot docker.io/pghalliday/tftp
```

This command will serve files from your current working directory via TFTP.
In addition, you can use Wireshark to monitor that interface to actually see what's happening.