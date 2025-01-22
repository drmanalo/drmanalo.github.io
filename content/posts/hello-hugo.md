+++
date = "2024-12-29T20:13:21Z"
draft = false
title = "Hello Hugo"
tags = ["golang","linux"]
+++`

## Prerequisites
Being an `archlinux` user means dependencies are for `arch` but this blog post should work with any `nix` distribution.
- [Linux](https://wiki.archlinux.org/title/Installation_guide)
- [Golang](https://go.dev/doc/install)

### Goodbye jbake
I was using `jbake` in the past as my static site generator. But since I'm using `golang` in my current role, I'm switching to `hugo` to create my blog.

```
❯ sudo pacman -S hugo
❯ hugo version
hugo v0.140.1+extended linux/amd64 BuildDate=unknown
```

### Create hugo site
There are steps 1 through 5 here that I'm keeping for documentation. That's pretty much you need to do as starter.
```
❯ hugo new site drmanalo
Congratulations! Your new Hugo site was created in ~/sunday.project/drmanalo.

Just a few more steps... 

1. Change the current directory to ~/sunday.project/drmanalo.
2. Create or install a theme:
   - Create a new theme with the command "hugo new theme <THEMENAME>"
   - Or, install a theme from https://themes.gohugo.io/
3. Edit hugo.toml, setting the "theme" property to the theme name.
4. Create new content with the command "hugo new content <SECTIONNAME>/<FILENAME>.<FORMAT>".
5. Start the embedded web server with the command "hugo server --buildDrafts".
```

### Step 1
```
❯ cd drmanalo
❯ git init
❯ git config user.email whoami@mailbox.com
❯ git config user.name drmanalo
❯ git add .
❯ git commit -m "hugo new site"
❯ git remote add origin git@github.com:drmanalo/drmanalo.github.io.git
❯ git branch -M main
❯ git push -u origin main
```

### Step 2
```
❯ git submodule add git@github.com:lukeorth/poison.git themes/poison
Cloning into '~/sunday.project/drmanalo/themes/poison'...
```

### Step 3
Edit `hugo.toml`
```
baseURL = "https://drmanalo.github.io/"
languageCode = "en-us"
title = "drmanalo"
theme = "poison"

[params]
    brand = "don"
    description = "Any fool can write code that a computer can understand"
    dark_mode = true

    menu = [
        {Name = "About", URL = "/about/", HasChildren = false},
        {Name = "Posts", URL = "/posts/", Pre = "Recent", HasChildren = true, Limit = 5},
    ]

    github_url = "https://github.com/drmanalo"
    linkedin_url = "https://linkedin.com/in/drmanalo"
    twitter_url = "https://x.com/drmanalo"
```

### Step 4
Then continue editing `content/posts/hello-hugo.md`
```
❯ hugo new content content/posts/hello-hugo.md
Content "~/sunday.project/drmanalo/content/posts/hello-hugo.md" created
```

### Step 5
Navigate to [http://localhost:1313](http://localhost:1313)
```
❯ hugo server -D
Start building sites … 
hugo v0.140.1+extended linux/amd64 BuildDate=unknown
                   | EN   
-------------------+------
  Pages            |   8  
  Paginator pages  |   0  
  Non-page files   |   0  
  Static files     | 161  
  Processed images |   0  
  Aliases          |   1  
  Cleaned          |   0  

Built in 27 ms
Environment: "development"
Serving pages from disk
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1) 
Press Ctrl+C to stop
```

### Github workflow
```
❯ mkdir -p .github/workflows
❯ touch .github/workflows/ci.yaml
```

### Edit `ci.yaml`
```
name: Deploy hugo to pages

on:
  push:
    branches:
      - main

  # Allows you to run this workflow manually from the Actions tab
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

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.137.1
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb          
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: Europe/London
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"          
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy as github pages
        id: deployment
        uses: actions/deploy-pages@v4
```        

### Watch github actions
```
❯ git add .
❯ git commit -m "github workflow"
❯ git push
```
