---
title: Getting Started with OSS EDA for ASIC Development 
author:
  - Johannes Pfau
description: How to synthesize simple digital designs for ASICs using OSS tools
ShowToc: true
series: ["OSS ASIC Design"]
categories: Hardware Development
tags:
  - OSS
  - OSS EDA
  - ASIC
  - ORFS
  - Distrobox
draft: false
---
Open source EDA tools and open source PDKs for various technologies have recently matured a lot.
Here's a simple and quick way to set up the development environment for those and get started with development.
<!--more-->

## Tool Overview

Digital ASIC development requires various tools (synthesis, implementation, ...) that are usually combined in a toolflow.
For an overview of commercial and OSS tools, see for example Mohammed Zakir Hussain's post [here](https://www.linkedin.com/posts/smzakirhussain_%F0%9D%90%83%F0%9D%90%9E%F0%9D%90%9A%F0%9D%90%AB-%F0%9D%90%9F%F0%9D%90%9E%F0%9D%90%A5%F0%9D%90%A5%F0%9D%90%A8%F0%9D%90%B0-%F0%9D%90%84%F0%9D%90%A5%F0%9D%90%9E%F0%9D%90%9C%F0%9D%90%AD%F0%9D%90%AB%F0%9D%90%A8%F0%9D%90%A7%F0%9D%90%A2%F0%9D%90%9C%F0%9D%90%AC-activity-7319178876700540929-3XNU?utm_source=share&utm_medium=member_desktop&rcm=ACoAAEOMOw0BO60MjGmgO8wqqMHhj3G3oo6z5Lw).
Make sure to check the updated table in the comments, as the original one has some errors.

In addition, you need the PDK (Process Design Kit) for your target technology.
PDKs are usually tool specific, so your PDK also needs explicit support for the OS tools.
Right now, such PDKs are available for GlobalFoundries [GF180MCU](https://github.com/google/gf180mcu-pdk), SkyWater [SKY130](https://skywater-pdk.readthedocs.io/en/main/) and IHP [SG13G2](https://github.com/IHP-GmbH/IHP-Open-PDK) BiCMOS SiGe technologies.

Setting up all tools and PDKs and updating them regularly is quite some effort.
Furthermore, if you want to do analog or mixed signal development, even more tools are needed.
The easiest way to install the tools is therefore using a prepared distribution.

A well-known, well-supported one is the [IIC-OSIC-TOOLS](https://github.com/iic-jku/IIC-OSIC-TOOLS/) container image.
Whereas the original installation instructions are fine, the setup can be simplified a lot on a Linux host when using [distrobox](https://distrobox.it).

## Distrobox Tool Installation

Setting up a distrobox container is quite simple:
```bash
distrobox create -i docker.io/hpretl/iic-osic-tools:2025.03 asic
distrobox enter asic
```

As the *IIC-OSIC-TOOLS* image does not [officially support distrobox](https://github.com/iic-jku/IIC-OSIC-TOOLS/issues/105) yet, we need to make some additional changes though:

```bash
# Use bash in the container, even if you use another shell in your host OS
chsh -s /bin/bash

# Automatically source the environment script when we enter the container
sudo ln -s /headless/.bashrc /etc/profile.d/z99_iic_osic_env.sh
sudo sh -c 'echo "source /headless/.bashrc" >> /etc/bash.bashrc'
```

## Running Graphical Tools

The *IIC-OSIC-TOOLS* image ships various tools, including grpahical tools for analog design.
As we're using distrobox, we can just run these tools from the container.
We don't have to use VNC, X11 forwarding or any other complex setup that is described in the *IIC-OSIC-TOOLS* documentation.

To test whether the tools are working, we can just run `klayout` once:

```bash
distrobox enter asic

# Use IHP PDK
iic-pdk ihp-sg13g2

# Try running klayout
klayout
```

## Synthesize a Digital Example Design with ORFS

Using the tools in the container, we can also run RTL2GDS flows for digital design.
Let's synthesize some Verilog example code for the IHP PDK using [OpenROAD Flow Scripts (ORFS)](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts).

Although *IIC-OSIC-TOOLS* ships OpenROAD tools and regularly tests ORFS, it does not ship ORFS as part of the container.
This is reasonable, as ORFS sometimes needs to be adapted to your design.
So you really want your own ORFS copy in a writable location.

First create a new project directory somewhere and clone ORFS:
```bash
distrobox enter asic
cd ~/Documents
git clone https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts.git orfs
cd orfs
```

{{< box warning >}}
It is [important](https://github.com/iic-jku/IIC-OSIC-TOOLS/issues/107) to use the ORFS commit matching the OpenROAD version shipped in the container.
If you use a newer version of ORFS, older OpenROAD tools in the container may just crash.
The `$TOOLS/openroad-latest/ORFS_COMMIT` file contains the ORFS git commit that was tested when assembling the container.
{{< /box >}}

Let's check out the tested ORFS revision:
```bash
git checkout $(cat $TOOLS/openroad-latest/ORFS_COMMIT)
```

With ORFS ready, we can now run a simple first synthesis.
According to the [test script](https://github.com/iic-jku/IIC-OSIC-TOOLS/blob/6b9fc3ba51b460852507564c7991543d52cf8eae/_tests/10/test_orfs_sg13g2.sh) in the *IIC-OSIC-TOOLS* repository, we need to set some tool paths:
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

We implicitly selected the PDK `ihp-sg13g2` and the design `spi` through the `DESIGN_CONFIG` path.
For information on how to use this with your own designs, see the ORFS [tutorial](https://openroad-flow-scripts.readthedocs.io/en/latest/tutorials/FlowTutorial.html) and [reference documentation](https://openroad-flow-scripts.readthedocs.io/en/latest/index2.html).

## VS Code Integration

For more convenient development, you'll probably want to use the container in VS Code as well.
This could mean running tools from the VS Code terminal or that your VS Code can use tools from the container.
For example, the [TerosHDL](https://terostechnology.github.io/terosHDLdoc/) extension needs a `yosys` executable, which can be used directly from this container.

{{< box warning >}}
When attaching to a container, VS Code by default connects as root user.
This will mess up file permissions: For distrobox containers, you should use the username matching your host OS.
Follow this guide completely and configure VS Code properly, **before** attaching to the container.
{{< /box >}}

Create the following configuration file at `$HOME/.config/Code/User/globalStorage/ms-VS Code-remote.remote-containers/nameConfigs/asic.json` to match the distrobox container named `asic`:
```json
{
	"remoteUser": "jpfau",
	"workspaceFolder": "/home/jpfau/Dokumente/Projekte/FPGA/neosd",
	"settings": {
		"terminal.integrated.defaultProfile.linux": "bash"
	}
}
```

`remoteUser` must match the username on your host OS.
`workspaceFolder` is optional and can be used to set a folder that will be opened by default.
As distrobox maps your home folder, you can choose any folder from your host OS as long as it's in your `$HOME`.
I'll also set `defaultProfile` to ensure that terminal windows use `bash` by default, as the container only sets up the environment for that shell.

{{< box info >}}
You should always start the container using `distrobox enter` before connecting in VS Code.
Don't let VS Code start the container. 
{{< /box >}}

You can now connect to the container in VS Code:
Press `Ctrl+Shift+P` and type `Dev Containers: Attach to running container`.

{{< box info >}}
For some reason, graphical applications don't seem to start from the VS Code terminal.
{{< /box >}}