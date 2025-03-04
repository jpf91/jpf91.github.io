---
title: Dev Containers Tricks for Distrobox
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

[Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) is a convenient way to develop in VS Code, whereas [distrobox](https://distrobox.it/) enables convenient GUI and device access in containers.
Both concepts can be combined using VS Code's *Attach to running container* command, but there are some things to remember.

<!--more-->

## Configuration for Attached Containers

Like for Dev Containers that are set up when you open a project, you can also specify `devcontainers.json` configuration when attaching to running containers.
The only difficulty here is locating the file though.
To find it, press `CTRL` + `SHIFT` + `p`, then execute the *Dev Containers: Open Attached Container Configuration File* command.

## Attach as Non-root User

By default, VS Code will attach to running containers as root user.
This will cause certain issues for distrobox containers, such as git complaining about incorrect file permissions.

To solve this, you can edit `devcontainer.json` [configuration for the attached container](#configuration-for-attached-containers) and add the `remoteUser` property.
When using distrobox, the `remoteUser` should be the username of your local user, as distrobox replicates that user in the container:
```json
{
	"workspaceFolder": "/home/jpfau/Dokumente/Projekte/FPGA/tang20k-neorv32",
	"settings": {
		"terminal.integrated.defaultProfile.linux": "bash"
	},
	"remoteUser": "jpfau"
}
```

In addition, you might want to also set the default terminal profile.
It seems to default to whatever is used on your host, but some container configurations might require specific shells.
As an example, the OSS FPGA Image I introduced in [a previous blog post]({{< relref "./2025-02-27-tang20k-neorv32.md" >}}) works best with the bash shell.

{{< box info >}}
If VS Code fails to connect to the container after changing the `remoteUser`, you probably attached to the container as root previously.
This installs `$HOME/.vscode-server/` with wrong file permissions, so delete that folder and then try to attach again.
{{< /box >}}