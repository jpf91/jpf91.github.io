---
title: Getting Started with ASIC Development and OSS Tools 
author:
  - Johannes Pfau
description: How to synthesize simple digital designs for ASICs using OSS tools
ShowToc: true
series: ["OSS ASIC Design"]
categories: Hardware Development
tags:
  - OSS
  - ASIC
draft: false
---
Recently, OSS tools and open source PDKs for various technologies have matured a lot.
Here's a simple and quick way on how to set up the development environment.
<!--more-->

## OSS Tool Overview

ASIC development requires various tools (synthesis, implementation, ...) that are usually combined in a toolflow (OpenROAD Flow Scripts, OpenLane).
In addition, you need the PDK (Process Design Kit) for your target technology.
PDKs are usually tool specific, so your PDK also needs explicit support for the OS tools.
Right now, such PDKs are available for GF180, SKY130 and IHP 130 technologies.

Setting up all those tools and PDKs, combining them and updating them regularly is quite some effort.
Additionally, if you want to do analog development or develop AMS systems, even more tools are needed.

The easiest way to install the tools is using a prepared distribution.
A well-known and well-supported one is the [IIC-OSIC-TOOLS](https://github.com/iic-jku/IIC-OSIC-TOOLS/) docker image.

Whereas the original installation instructions are ok, when using a Linux host, [distrobox](https://distrobox.it) can simplify things a lot.

## Distrobox Tool Installation

Setting up a distrobox container is quite simple:
```bash
distrobox create -i docker.io/hpretl/iic-osic-tools:2025.03 asic
distrobox enter asic
```

As the *IIC-OSIC-TOOLS* image does not [officially support distrobox](https://github.com/iic-jku/IIC-OSIC-TOOLS/issues/105) yet, we need to make some additional changes:

```bash
# Use bash in the container, even if you use another shell
# in your host OS
chsh -s /bin/bash

# Automatically source the environment script when we enter the container
sudo ln -s /headless/.bashrc /etc/profile.d/z99_iic_osic_env.sh
sudo sh -c 'echo "source /headless/.bashrc" >> /etc/bash.bashrc'
```

## Running Graphical Tools

The *IIC-OSIC-TOOLS* image ships various tools, including tools for analog design.
As we're using distrobox, we can just run these tools in the container.
We therefore don't need to use VNC, X11 forwarding or any other complex setup that is described in the *IIC-OSIC-TOOLS* documentation.

To test whether the tools are working, we can just run `klayout` once:

```bash
distrobox enter asic

# Use IHP PDK
iic-pdk ihp-sg13g2

# Try running klayout
klayout
```

## Synthesize a Digital Example Design with ORFS

We can now also use the RTL2GDS flow for digital design.
Let's have a look at how we can synthesize Verilog code for the IHP SDK using OpenROAD Flow Scripts (ORFS).

Although *IIC-OSIC-TOOLS* ships OpenROAD and regularly tests ORFS, it does not ship ORFS in the container.
This is reasonable, as ORFS sometimes needs to be modified, so you want a copy in a writable location.

So first create a new project directory and clone ORFS:
```bash
distrobox enter asic
cd Documents
git clone https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts.git orfs
cd orfs
```

It is [quite important](https://github.com/iic-jku/IIC-OSIC-TOOLS/issues/107) to use the ORFS commit matching the OpenROAD tools shipped in the container.
If you use a newer version of ORFS, the old OpenROAD tools in the container may crash.
The *$TOOLS/openroad-latest/ORFS_COMMIT* file contains the has of the ORFS version that was tested when assembling the container.
We should therefore use this revision:
```bash
git checkout $(cat $TOOLS/openroad-latest/ORFS_COMMIT)
```

With ORFS prepared, we can now run a simple first synthesis.
According to the [test script](https://github.com/iic-jku/IIC-OSIC-TOOLS/blob/6b9fc3ba51b460852507564c7991543d52cf8eae/_tests/10/test_orfs_sg13g2.sh) in the *IIC-OSIC-TOOLS* repository, we should set some tool paths:
```bash
export YOSYS_EXE=$TOOLS/yosys/bin/yosys
export OPENROAD_EXE=$TOOLS/openroad-latest/bin/openroad
export OPENSTA_EXE=$TOOLS/openroad-latest/bin/sta
export FLOW_HOME
```

Then enter the flow directory, select an example to synthesize and run the flow:
```bash
cd flow
export DESIGN_CONFIG=./designs/ihp-sg13g2/spi/config.mk
make
```

Here we implicitly selected the PDK (`ihp-sg13g2`) and the design (`spi`).

## VS Code Integration

For more convenient development, you probably want to use the container in VS Code as well.
In the simplest case this means running tools from the VS Code terminal.
It however also means that your VS Code can use tools from the container.
For example, the [Teros HDL]() extension needs a `yosys` executable, which can be used directly from this container.

{{< box warning >}}
When attaching to a container, VS Code by default uses the root user.
This will mess up file permissions, as in distrobox you should use the user with the same name as on your host OS.
Before you attach to the container, follow this guide fully to configure VS Code properly.
{{< /box >}}

Add the following configuration for the container named `asic`, located at `$HOME/.config/Code/User/globalStorage/ms-VS Code-remote.remote-containers/nameConfigs/asic.json`:
```json
{
	"remoteUser": "jpfau",
	"workspaceFolder": "/home/jpfau/Dokumente/Projekte/FPGA/neosd",
	"settings": {
		"terminal.integrated.defaultProfile.linux": "bash"
	}
}
```

Here `remoteUser` must match the username on your host OS.
The `workspaceFolder` key is optional and can be used to configure the folder that will be opened by default when attaching to the container.
As distrobox maps your complete home folder, you can choose any folder from your host OS if it's in your `$HOME`.
We'll also set `defaultProfile` to ensure that terminal windows use `bash` by default, as we only set up the environment for that shell.

You can now connect to the container in VS Code:
Press `Ctrl+Shift+P` and type `Dev Containers: Attach to running container`.

{{< box info >}}
For some reason, graphical applications don't seem to start from the VS Code terminal.
{{< /box >}}

{{< box info >}}
You should always start the container using `distrobox start` before connecting in VS Code.
Don't let VS Code start the container. 
{{< /box >}}