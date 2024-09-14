# jpfau.org sources

The [jpfau.org](jpfau.org) website is built using the [vuepress](https://vuepress.vuejs.org/) static site generator and the [hope theme](https://theme-hope.vuejs.press/). 
Documentation on supported writing features can be found here: https://theme-hope.vuejs.press/get-started .

In order to handle the node dependency mess, all commands are executed in a docker container. Currently, the node 22.8.0 container is used.

## Editing content in VSCode

### Setup
First install [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers), then set it up for podman, i.e. use these settings:
```json
"dev.containers.dockerPath": "podman"
"dev.containers.dockerSocketPath": "/run/user/1000/podman/podman.sock"
```

Note: When setting up a new website, you'll have to setup `.devcontainer/devcontainers.json`.
You can use the one included in this repository as a reference.

### Usage

* Use Ctrl-Shift-P and the "Dev Containers: Open Folder in Container" command.
* Select your `jpf91.github.io` clone.
* Wait for the container to load.
* If you use multiple monitors, just open your browser on another screen and open [http://localhost:8080/](http://localhost:8080/).
* If you use only one monitor, you can also preview the site in VSCode:
  * Use Ctrl-Shift-P and the "Simple Browser: Show" command.
  * Enter `http://localhost:8080/` as URL.
  * You can drag the tab to the window edge for a split editor / preview setup.

## Manual CLI commands

### Initializing the site

These commands were used to generate the initial source directory:
```bash
podman run -p 8080:8080 -v .:/data:Z -it --rm docker.io/node:22.8.0-slim /bin/bash
cd /data
corepack enable
pnpm create vuepress-theme-hope jpfau.org
```

### Previewing the site locally
```bash
podman run -p 8080:8080 -v .:/data:Z -it --rm docker.io/node:22.8.0-slim /bin/bash
cd /data/jpfau.org
pnpm docs:clean-dev
```

The preview is at [http://localhost:8080/](http://localhost:8080/).

### Building the site
```bash
podman run -p 8080:8080 -v .:/data:Z -it --rm docker.io/node:22.8.0-slim /bin/bash
cd /data/jpfau.org
pnpm docs:build
```

Generated files are in `src/.vuepress/dist/`.

## Updating the theme
```bash
podman run -p 8080:8080 -v .:/data:Z -it --rm docker.io/node:22.8.0-slim /bin/bash
cd /data/jpfau.org
pnpm dlx vp-update
```