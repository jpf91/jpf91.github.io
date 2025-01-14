---
title: Setting up a Hugo Blog
author:
  - Johannes Pfau
description: A quick introduction to set up Hugo with PaperMod theme
tags:
  - Hugo
  - IT
ShowToc: true
draft: false
series:
  - Blogging with Hugo
---
Setting up a static HTML based blog can be a bit challenging, and deciding which software and theme to use even more!
Hugo is a widely used and well-supported solution with some great themes.
So here's a quick start guide on how to set up a good-looking Hugo blog!

<!--more-->

## Configuring Dev Containers

First, we have will have to install Hugo.
Using Hugo in a container is a good idea to make sure we don't clutter the host OS, to make our installation portable and also make sure we can use a recent Hugo version on older OS.
A simple way to use containers and get good integration into Visual Studio Code is using its [Dev Containers](https://code.visualstudio.com/docs/devcontainers/tutorial) feature.

As a first step, let's create a new git repository where we will place all data and create the folder for the Dev Container configuration:

```bash
git init jpfau.org
mkdir jpfau.org/.devcontainer
```

Next, let's create the `.devcontainer/devcontainer.json` file with this content:

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the
// README at: https://github.com/devcontainers/templates/tree/main/src/javascript-node
{
  "name": "jpfau.org",
  "build": {
    "dockerfile": "Dockerfile",
    "args": {
      "HUGO_VERSION": "0.140.2"
    }
  },
  "forwardPorts": [1313],
  "customizations": {
    "vscode": {
      "extensions": [
        "valentjn.vscode-ltex",
        "eamodio.gitlens",
        "eliostruyf.vscode-front-matter"
      ]
    }
  }
}
```

This will instruct VS Code to build a container using the local `Dockerfile`, pass `HUGO_VERSION` as a build variable, forward port 1313 when running the container and install the listed VS Code extensions.
Even when editing markdown instead of latex, `valentjn.vscode-ltex` will be useful for spell and grammar checking. `eamodio.gitlens` provides extended GIT support and `eliostruyf.vscode-front-matter` acts as a CMS VS Code add-on.

Although there are ready-to-use Hugo containers from the HugoMods project, those can't run the `LanguageTool` server used in LTeX.
Furthermore, those images don't include SSH support for GIT.
Because of this, we will build a custom image based on Ubuntu and the latest official Hugo binary releases.
Add the following to `.devcontainer/Dockerfile`:

```Dockerfile
FROM docker.io/ubuntu:24.04
ARG HUGO_VERSION

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get -qq -y install curl git golang-go \
    && apt-get clean
RUN curl -sL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb" -o hugo.deb && \
    dpkg -i hugo.deb && \
    rm hugo.deb
```

This will use the latest Ubuntu LTS release, install some dependencies and finally will install the official Hugo binaries.
The Hugo version can be specified as a build variable.

## Installing the PaperMod Theme

Open the newly created folder in VS Code and let it set up a container.
Once the container has been built and opened, open a terminal in VS Code.
Then initialize a new Hugo site:

```bash
hugo new site jpfau.org --format yaml
```

Next, follow the [PaperMod install instructions](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation) and set up the git submodule:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

You might want to set up `.gitignore` to ignore built files:

```
/public/
.hugo_build.lock
```

Then, to get the basic site ready, create `content/archive.md`:

```markdown
---
title: "Archive"
layout: "archives"
summary: "archive"
---
```

and `content/search.md`:

```markdown
---
title: "Search"
placeholder: Site Search
layout: "search"
---
```

Finally, modify your `hugo.yaml` to look like this:

```yaml
baseURL: https://jpfau.org/
languageCode: en-us
title: JP's Techblog
theme:
  - PaperMod
enableInlineShortcodes: true
enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false
enableEmoji: true
pygmentsUseClasses: true
pluralizelisttitles: false
mainsections: ["blog"]

frontmatter:
  date:
    - ':filename'
    - ':default'

languages:
  en:
    languageName: "English"
    weight: 1
    taxonomies:
      category: categories
      tag: tags
      series: series
    menu:
      main:
        - name: Series
          url: series/
          weight: 5
        - name: Archive
          url: archive/
          weight: 5
        - name: Search
          url: search/
          weight: 10
        - name: Tags
          url: tags/
          weight: 10

outputs:
  home:
  - HTML
  - RSS
  - JSON

params:
  env: production
  defaultTheme: auto
  ShowShareButtons: true
  ShowReadingTime: true
  displayFullLangName: true
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowRssButtonInSectionTermList: true
  ShowAllPagesInArchive: true
  ShowPageNums: true
  ShowToc: true

  homeInfoParams:
    Title: "Johannes Pfau"
    Content: >
      Blogging about all things tech, software, hardware and IT stuff.
  socialIcons:
    - name: github
      title: Github
      url: "https://github.com/jpf91"
    - name: LinkedIn
      title: LinkedIn
      url: "https://linkedin.com/in/johannes-pfau-a3963b277"
    - name: ORCID
      title: ORCID
      url: "https://orcid.org/0000-0003-1087-1814"
```

The `frontmatter` snippet ensures that dates are [read from filenames](https://discourse.gohugo.io/t/have-date-in-the-filepath-but-not-in-the-urls/42650), if you name your files `YYYY-MM-DD-blog-name.md`.
The `mainsections` configuration tells Hugo that it should look for posts in `content/blog`.
Refer to the [PaperMod Example Site](https://github.com/adityatelange/hugo-PaperMod/tree/exampleSite) for more configuration options.

## Previewing the Site

You can now preview the site by executing `hugo serve --bind 0.0.0.0`.
When the server is running, open `localhost:1313` in your browser, or press Ctrl-Shift-P and type `Simple browser show` to open a web browser in VS Code.

To add articles, simply place them in `content/blog` and name them `YYYY-MM-DD-blog-name.md`, e.g. `2025-01-14-setting-up-hugo.md`.

## Deploying with GitHub Actions

To build using GitHub Actions and deploy to GitHub Pages, create `.github/workflows/site.yaml` with this content:

```yaml
name: site

on:
  # trigger deployment on every push to main branch
  push:
    branches: [master]
  # trigger deployment manually
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  docs:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
        with:
          # fetch all commits to get last updated time or other git log info
          fetch-depth: 0
          submodules: true

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.140.2'
          extended: true

      # Build
      - name: Build Hugo site
        run: hugo --minify

      # Deploy to github pages
      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'public'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Next Steps

In the next posts, we'll see how to add comments, how to install DecapCMS, how to set up Latex, how to get article series working and how to get info boxes and other markdown enhancements.
