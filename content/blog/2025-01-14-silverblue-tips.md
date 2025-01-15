---
title: Fedora Silverblue Tips
author:
  - Johannes Pfau
description: A list of useful tips when using Silverblue
tags:
  - Fedora
  - IT
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

