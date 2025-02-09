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


{{< box warning >}}
  Docker in the container seems to work only if SELinux is disabled on the host.
{{< /box >}}

## Adjusting the Flow

Once you have the flow running locally, you can now look at reports and adjust the flow to your needs.
Behind the scenes, Tiny Tapeout uses the usual OpenLane 2 flow, so output files and most configuration will follow the usual OpenLane setup.
For details, refer to the [OpenLane documentation](https://openlane2.readthedocs.io/en/latest/getting_started/newcomers/index.html).


To give you a head start, one of the first things you may want to do is check the size of your design.
All output files, reports and command logs are available in the `runs/wokwi/` folder.
So for area usage, let's have a look at `27-openroad-globalplacement/openroad-globalplacement.log`.
The relevant part of the file looks like this:
```
[INFO GPL-0006] NumInstances:              1324
[INFO GPL-0007] NumPlaceInstances:         1021
[INFO GPL-0008] NumFixedInstances:          303
[INFO GPL-0009] NumDummyInstances:            0
[INFO GPL-0010] NumNets:                   1040
[INFO GPL-0011] NumPins:                   3459
[INFO GPL-0012] DieBBox:  (  0.000  0.000 ) ( 161.000 111.520 ) um
[INFO GPL-0013] CoreBBox: (  2.760  2.720 ) ( 158.240 108.800 ) um
[INFO GPL-0016] CoreArea:             16493.318 um^2
[INFO GPL-0017] NonPlaceInstsArea:      574.301 um^2
[INFO GPL-0018] PlaceInstsArea:       11977.738 um^2
[INFO GPL-0019] Util:                    75.242 %
[INFO GPL-0020] StdInstsArea:         11977.738 um^2
[INFO GPL-0021] MacroInstsArea:           0.000 um^2
```

Here `Util` is probably the main measurement you're looking for.

To customize the Tiny Tapeout setup (for example to set the number of tiles used), edit `info.yaml`.
Refer to the comments in the [info.yaml file](https://github.com/TinyTapeout/tt10-verilog-template/blob/main/info.yaml) for more information on available options.

For more complex customization, you can directly adjust the OpenLane configuration in the `src/config.json` file.
If you do, you can refer to these documentation sites:
* [OpenLane Getting Started](https://openlane2.readthedocs.io/en/latest/getting_started/newcomers/index.html)
* [OpenLane Config Variables](https://openlane2.readthedocs.io/en/latest/reference/step_config_vars.html)
* [OpenLane Config Variables](https://openlane2.readthedocs.io/en/latest/reference/step_config_vars.html)
* [OpenLane Timing Closure](https://openlane2.readthedocs.io/en/latest/usage/timing_closure/index.html)
* [OpenRoad Main Documentation](https://openroad.readthedocs.io/en/latest/main/src/README.html)
* [OpenRoad Clock-Tree Synthesis](https://openroad.readthedocs.io/en/latest/main/src/cts/README.html)

In a future blog post, I will summarize common options and provide some examples for common adjustments.