+++
date = "2025-04-21T20:58:58+01:00"
draft = true
title = "Portainer on Mac"
+++

## Prerequisites
Being a `Macbook` user means dependencies are for `brew` but this blog post should work with any `unix` distribution.
- [homebrew](https://brew.sh/)


## Install podman
```
❯ brew install podman
==> Downloading https://ghcr.io/v2/homebrew/core/podman/manifests/5.4.2
==> Fetching podman
==> Downloading https://ghcr.io/v2/homebrew/core/podman/blobs/sha256:ca637f208cb8fb208c8552063d0e5021809
==> Pouring podman--5.4.2.arm64_sequoia.bottle.tar.gz
==> Caveats
In order to run containers locally, podman depends on a Linux kernel.
One can be started manually using `podman machine` from this package.
To start a podman VM automatically at login, also install the cask
"podman-desktop".
```
You can follow the recommendation to install `podman-desktop` so you can start podman VM automatically but I'm choosing `launchctl` since I'm going to use portainer later on.

## Initialise podman
```
❯ podman machine init --cpus 2 --disk-size 100 --memory 4096
Looking up Podman Machine image at quay.io/podman/machine-os:5.4 to create VM
Getting image source signatures
Copying blob 49c3b3279cd1 done   |
Copying config 44136fa355 done   |
Writing manifest to image destination

❯ podman machine set --rootful

❯ sudo /opt/homebrew/Cellar/podman/your.version/bin/podman-mac-helper install

❯ podman machine start
Starting machine "podman-machine-default"
API forwarding listening on: /var/run/docker.sock
Docker API clients default to this address. You do not need to set DOCKER_HOST.

Machine "podman-machine-default" started successfully
```    

## Podman machine launcher

Create this file called `com.user.podman-machine.plist` inside `~/Library/LaunchAgents` then paste the XML below.
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>KeepAlive</key>
	<true/>
	<key>Label</key>
	<string>homebrew.mxcl.podman</string>
	<array>
        <string>/opt/homebrew/bin/podman</string>
        <string>machine</string>
        <string>start</string>
    </array>
	<key>RunAtLoad</key>
	<true/>
	<key>StandardOutPath</key>
    <string>/tmp/podman-machine-start.out</string>
    <key>StandardErrorPath</key>
    <string>/tmp/podman-machine-start.err</string>
</dict>
</plist>
```

You can logout then login to confirm it works, or manually unload and reload the agent.
```
❯ launchctl load ~/Library/LaunchAgents/com.user.podman-machine.plist
❯ podman machine ls
NAME                    VM TYPE     CREATED         LAST UP             CPUS        MEMORY      DISK SIZE
podman-machine-default  applehv     15 minutes ago  Currently starting  7           2GiB        100GiB
❯ launchctl unload ~/Library/LaunchAgents/com.user.podman-machine.plist
```
