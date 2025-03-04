---
title: Fedora Silverblue Tips
author:
  - Johannes Pfau
description: A list of useful tips when using Silverblue
series: ["Container-Based OS and Development"]
tags:
  - Fedora
  - IT
categories: Desktop Linux
draft: false
---

All in all, Fedora Silverblue is easy to use.
Some things are however still more complex in Silverblue, so this post tries to give some useful hints.

<!--more-->

## Visual Studio Code

Visual Studio Code can be installed from Flathub as usual.
However, getting the Dev Container support working requires some more work.
As of January 2025, [this](https://gist.github.com/queeup/1c9713745be5ef7ce426795045a9ae9f) seems to be the most up-to-date tutorial.
To summarize:
* Install VS Code from Flathub.
* Allow access to `/tmp`, needed to build containers:
  ```bash
  flatpak override --user --filesystem=/tmp com.visualstudio.code
  ```
* Create the `~/.local/bin/podman-host` wrapper.
* In the VS Code Settings, set `Docker Path` to that wrapper.

[Here's](https://github.com/theonlyfoxy/my-devcontainers/tree/main/Embedded) an advanced example with USB and GPU pass-through and X11 support.

{{< box info >}}
  If VS Code opens the folder in the container, but you don't see any files, and if `ls` in the terminal responds with "Permission denied", this is likely a SELinux issue.
  [Here's](https://github.com/microsoft/vscode-remote-release/issues/1333#issuecomment-702568494) a long-standing issue report with some workarounds.
  For temporal testing, you can see if everything works with SELinux disabled: `sudo setenforce 0`.
  If it does, adding this to your `devcontainer.json` might work as a less intrusive workaround:
  ```json
  "runArgs": [
    "--userns=keep-id",
    "--security-opt=label=disable"
  ],
  ```
{{< /box >}}

## Saleae Logic

As on any linux system, you'll first have to install the udev rules.
Add the following to `/etc/udev/rules.d/99-SaleaeLogic.rules`:
```
#
# Saleae Logic
# This file should be installed to /etc/udev/rules.d so that you can access the Saleae Logic hardware without being root
#
# Type this at the command prompt: sudo cp 99-SaleaeLogic.rules /etc/udev/rules.d
#

SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="0925", ATTR{idProduct}=="3881", MODE="0666"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="21a9", ATTR{idProduct}=="1001", MODE="0666"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="21a9", ATTR{idProduct}=="1003", MODE="0666"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="21a9", ATTR{idProduct}=="1004", MODE="0666"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="21a9", ATTR{idProduct}=="1005", MODE="0666"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="21a9", ATTR{idProduct}=="1006", MODE="0666"
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="21a9", ATTR{idProduct}=="1007", MODE="0666"
```

In addition, the Saleae Logic 2 software uses some libraries that are not installed by default on Silverblue.
If you run the AppImage file directly, you will therefore just get a `Error Connecting to Socket` message.

To solve this, you need to install the `libnsl` and `libxcrypt-compat` packages.
Unfortunately, I couldn't get the Saleae software to work in a [distrobox](https://distrobox.it) container, so for now it seems the packages need to be layered:
```bash
sudo rpm-ostree install --apply-live libnsl libxcrypt-compat
```

Let's hope Saleae builds a Flatpak bundle for Flathub at some point, to make installation of the software much easier.