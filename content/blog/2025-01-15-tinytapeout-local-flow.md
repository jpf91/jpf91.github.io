---
title: Running the TinyTapeout Flow Locally
author:
  - Johannes Pfau
description: How to run the TinyTapeout flow on your PC instead of in the cloud
tags:
  - OSS EDA
  - Tiny Tapeout
ShowToc: true
categories: Hardware Development
draft: false
---
Tiny Tapeout has a [local hardening guide](https://tinytapeout.com/guides/local-hardening/) to build your design locally.
However, the guide does not list required system dependencies and is a bit difficult to follow.
This guide aims to be more complete.

<!--more-->

{{< box info >}}
  This was last tested for the [TT10 shuttle](https://app.tinytapeout.com/shuttles/tt10).
{{< /box >}}

## Setting up a Container for TT

The [local hardening guide](https://tinytapeout.com/guides/local-hardening/) assumes that we have a local linux OS with certain dependencies installed.
In order to be independent of the host OS, I prefer to build in a container.
In this case this is slightly more complicated, as the flow requires a working docker daemon and graphical output.
To satisfy these requirements, we'll set up a container in [distrobox](https://distrobox.it/) (with podman backend) that supports running docker.

First, temporarily disable SELinux, as the installation of some packages fails otherwise:
```bash
sudo setenforce 0
```

Then create a distrobox container [that supports running docker](https://github.com/89luca89/distrobox/blob/main/docs/useful_tips.md#using-docker-inside-a-distrobox):
```bash
distrobox create --image fedora:41 --additional-packages "systemd docker" --init --unshare-all tt
distrobox enter tt
```

Next, enable docker in the container and add our user to the docker group:
```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
exit
```

We have to exit the container and enter again to make sure the group information is updated.
After that, we should be able to use docker:
```bash
distrobox enter tt
docker info

Client:
 Version:    27.3.1
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  0.18.0
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
```

## Installing Dependencies

Now let's install the dependencies for the TT flow in the container:
```bash
sudo dnf group install c-development development-tools
sudo dnf install python3-tkinter python3-devel swig

# Install NIX
sh <(curl -L https://nixos.org/nix/install)
```

## Following the Hardening Guide

We can now follow the [local hardening guide](https://tinytapeout.com/guides/local-hardening/):

```bash
git clone https://github.com/TinyTapeout/tt10-factory-test factory-test
git clone -b tt10 https://github.com/TinyTapeout/tt-support-tools factory-test/tt

mkdir ttsetup
python3 -m venv ttsetup/venv
source ttsetup/venv/bin/activate
pip install -r factory-test/tt/requirements.txt

export PDK_ROOT=$PWD/ttsetup/pdk
export PDK=sky130A
export OPENLANE2_TAG=2.1.9
```

{{< box warning >}}
  At least for me, installation of `libparse` always failed when simply following the guide.
  After manual installation, the rest of the process works fine:
  ```bash
  git clone --recursive --branch 0.3.1 https://github.com/efabless/libparse-python.git
  pip install libparse-python/
  rm -rf libparse-python
  ```
{{< /box >}}

Now we can install the remaining python packages:
```bash
pip install openlane==$OPENLANE2_TAG
```

And now we're finally ready to build the project:
```bash
source $HOME/.nix-profile/etc/profile.d/nix.sh
cd factory-test
./tt/tt_tool.py --create-user-config --openlane2
./tt/tt_tool.py --harden --openlane2
./tt/tt_tool.py --print-warnings --openlane2
```