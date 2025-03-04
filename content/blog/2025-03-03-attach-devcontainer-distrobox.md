---
title: DevContainer Tricks for Distrobox
author:
  - Johannes Pfau
description: Some interesting points when using DevContainers with distrobox.
series: ["Container-Based OS and Development"]
ShowToc: true
categories: Desktop Linux
tags:
  - Distrobox
draft: false
---

[DevContainers](https://code.visualstudio.com/docs/devcontainers/containers) is a convenient way to develop in VS Code, whereas [distrobox](https://distrobox.it/) enables convenient GUI and device access in containers.
Both concepts can be combined using VS Code's *Attach to running container* command, but there are some things to remember.

<!--more-->

## Configuration for Attached Containers

Just like for Devcontainers that are created when you open a project, you can also specify a `devcontainers.json` configuration that is used when attaching to a running container.
The main difficulty here is locating the file:
Simply press `CTRL` + `SHIFT` + `p` and execute the *Dev Containers: Open Attached Container Configuration File* command to open the right file.

## Attach as Non-root User

By default, VS Code will attach as root user to running containers.
This will cause certain issues for distrobox containers, for example the git window integration complaining about incorrect file permissions etc.

To solve this, you can edit the `devcontainer.json` [configuration for attached containers](#configuration-for-attached-containers) and add the `remoteUser` property:
```json
{
	"workspaceFolder": "/home/jpfau/Dokumente/Projekte/FPGA/tang20k-neorv32",
	"settings": {
		"terminal.integrated.defaultProfile.linux": "bash"
	},
	"remoteUser": "jpfau"
}
```

In addition, you might also want to set the terminal profile.
The terminal seems to default to whatever is used on your host, but
some container configurations might require specific shells.
For example, the OSS FPGA Image I introduced in [a previous blog post]({{< relref "./2025-02-27-tang20k-neorv32.md" >}}) works best with the bash shell.

{{< box info >}}
If VS Code fails to connect to the container after changing the `remoteUser`, you probably attached to the container as root previously.
This installs `$HOME/.vscode-server/` with wrong file permissions, so delete that folder and then try to attach again.
{{< /box >}}