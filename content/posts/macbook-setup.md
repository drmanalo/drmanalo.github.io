+++
date = "2025-03-02T14:09:13Z"
draft = false
title = "Macbook Setup"
tags = ["macbook","ansible"]
+++

## Prerequisites
Being a `Macbook` user means dependencies are for `brew` but this blog post should work with any `unix` distribution.
- [pip3](#setting-up-mac)

## Goodbye hyprland
It is a weird feeling but I have to say goodbye to my `archlinux`. From my previous posts, I've been a Linux user and enjoying every bit of it. But now I have MacBook M4 which feels faster than my miniPC so I decided to make a switch. There is a learning curve to retrain my fingers for typing and for using gestures. It pains me to say goodbye to my tiling window application `hyprland` and I'm still searching for the closest replacement to it.

## Setting up Mac
  1. Ensure Apple's command line tools are installed.`xcode-select --install` to launch the installer.
  1. [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/index.html):

     1. Run the following command to add Python 3 to your $PATH: `export PATH="$HOME/Library/Python/3.9/bin:/opt/homebrew/bin:$PATH"`
     1. Upgrade Pip: `sudo pip3 install --upgrade pip`
     1. Install Ansible: `pip3 install ansible`

  1. Clone [this repository](https://github.com/drmanalo/macsetup) to your local drive.
  1. Run `ansible-galaxy install -r requirements.yml` inside this directory to install required Ansible roles.
  1. Run `ansible-playbook main.yml --ask-become-pass` inside this directory. Enter your macOS account password when prompted for the 'BECOME' password.

  ## Install using remote machine
  You can use the playbook from my repository to manage remote Macs as well. The playbook doesn't even need to be run from a Mac at all! You just need to make sure you can connect to it with SSH:

  1. On the Mac you want to connect to go to System Preferences > Sharing.
  1. Enable 'Remote Login'.

You can also enable remote login on the command line:

>     sudo systemsetup -setremotelogin on

Then edit the `inventory` file in this repository and change the line that starts with `127.0.0.1` to:

```
[ip address or hostname of mac]  ansible_user=ssh
```

And copy your SSH key to the remote Mac using 
```
ssh-copy-id user@remote.mac
```

## Included Applications by default
Edit `default.config.yml` if you don't like the applications below.

### Applications with Homebrew Cask
  - [brave-browser](https://brave.com/)
  - [chromedriver](https://sites.google.com/chromium.org/driver/)
  - [firefox](https://www.mozilla.org/en-US/firefox/new/)
  - [font-meslo-lg-nerd-font](https://github.com/ryanoasis/nerd-fonts)
  - [google-chrome](https://www.google.com/chrome/)
  - [intellij-idea-ce](https://www.jetbrains.com/idea/)
  - [iterm2](https://iterm2.com/)
  - [kitty](https://github.com/kovidgoyal/kitty)
  - [lm-studio](https://lmstudio.ai/)
  - [licecap](http://www.cockos.com/licecap/)
  - [nikitabobko/tap/aerospace](https://github.com/nikitabobko/AeroSpace)
  - [sequel-ace](https://github.com/Sequel-Ace/Sequel-Ace)
  - [slack](https://slack.com/)
  - [sublime-text](https://www.sublimetext.com/)
  - [tradingview](https://www.tradingview.com/desktop/)
  - [trilium-notes](https://github.com/zadam/trilium)
  - [visual-studio-code](https://code.visualstudio.com/)

### Packages installed using homebrew
  - autoconf
  - bat
  - doxygen
  - eza
  - fzf
  - delta
  - gettext
  - git
  - gh
  - go
  - gpg
  - hashicorp/tap/terraform
  - httpie
  - hugo
  - iperf
  - libevent
  - mockery
  - neovim
  - nmap
  - node
  - nvm
  - ollama
  - pass
  - podman
  - powerlevel10k
  - pngpaste
  - ssh-copy-id
  - readline
  - openssl
  - pv
  - speedtest-cli
  - sqlite
  - thefuck
  - tldr
  - wget
  - wrk
  - zsh-autosuggestions
  - zsh-autocomplete

## Downloading the roles
```
❯ ansible-galaxy install -r requirements.yml
Starting galaxy role install process
- downloading role 'osx-command-line-tools', owned by elliotweiser
- downloading role from https://github.com/elliotweiser/ansible-osx-command-line-tools/archive/2.3.0.tar.gz
- extracting elliotweiser.osx-command-line-tools to ~/sunday.project/macbook-setup/roles/elliotweiser.osx-command-line-tools
- elliotweiser.osx-command-line-tools (2.3.0) was installed successfully
- downloading role 'dotfiles', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-dotfiles/archive/1.3.2.tar.gz
- extracting geerlingguy.dotfiles to ~/sunday.project/macbook-setup/roles/geerlingguy.dotfiles
- geerlingguy.dotfiles (1.3.2) was installed successfully
Starting galaxy collection install process
Nothing to do. All requested collections are already installed. If you want to reinstall them, consider using `--force`.
```

## Running the playbook
```
❯ ansible-playbook -K main.yml
BECOME password:

PLAY [Configure host.] *****************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************************************************************
[WARNING]: Platform darwin on host 192.168.0.59 is using the discovered Python interpreter at /usr/bin/python3, but future installation of another Python interpreter could change the meaning of that path. See
https://docs.ansible.com/ansible-core/2.15/reference_appendices/interpreter_discovery.html for more information.
ok: [192.168.0.59]

TASK [Include playbook configuration.] *************************************************************************************************************************************************************************************
skipping: [192.168.0.59]

...

TASK [geerlingguy.mac.homebrew : Install configured cask applications.] ****************************************************************************************************************************************************
changed: [192.168.0.59] => (item=anythingllm)
changed: [192.168.0.59] => (item=brave-browser)
changed: [192.168.0.59] => (item=chromedriver)
changed: [192.168.0.59] => (item=firefox)
changed: [192.168.0.59] => (item=font-meslo-lg-nerd-font)
changed: [192.168.0.59] => (item=google-chrome)
changed: [192.168.0.59] => (item=intellij-idea-ce)
changed: [192.168.0.59] => (item=iterm2)
changed: [192.168.0.59] => (item=kitty)
changed: [192.168.0.59] => (item=licecap)
changed: [192.168.0.59] => (item=nikitabobko/tap/aerospace)
changed: [192.168.0.59] => (item=sequel-ace)
changed: [192.168.0.59] => (item=slack)
changed: [192.168.0.59] => (item=sublime-text)
changed: [192.168.0.59] => (item=tradingview)
changed: [192.168.0.59] => (item=trilium-notes)
changed: [192.168.0.59] => (item=visual-studio-code)

TASK [geerlingguy.mac.homebrew : Ensure blacklisted homebrew packages are not installed.] **********************************************************************************************************************************
skipping: [192.168.0.59]

TASK [geerlingguy.mac.homebrew : Ensure configured homebrew packages are installed.] ***************************************************************************************************************************************
changed: [192.168.0.59] => (item=autoconf)
changed: [192.168.0.59] => (item=bat)
changed: [192.168.0.59] => (item=doxygen)
changed: [192.168.0.59] => (item=eza)
changed: [192.168.0.59] => (item=fzf)
changed: [192.168.0.59] => (item=delta)
changed: [192.168.0.59] => (item=gettext)
changed: [192.168.0.59] => (item=git)
changed: [192.168.0.59] => (item=gh)
changed: [192.168.0.59] => (item=go)
changed: [192.168.0.59] => (item=gpg)
changed: [192.168.0.59] => (item=hashicorp/tap/terraform)
changed: [192.168.0.59] => (item=httpie)
changed: [192.168.0.59] => (item=iperf)
ok: [192.168.0.59] => (item=libevent)
changed: [192.168.0.59] => (item=mockery)
changed: [192.168.0.59] => (item=neovim)
changed: [192.168.0.59] => (item=nmap)
changed: [192.168.0.59] => (item=node)
changed: [192.168.0.59] => (item=nvm)
changed: [192.168.0.59] => (item=ollama)
changed: [192.168.0.59] => (item=pass)
changed: [192.168.0.59] => (item=podman)
changed: [192.168.0.59] => (item=powerlevel10k)
changed: [192.168.0.59] => (item=pngpaste)
changed: [192.168.0.59] => (item=ssh-copy-id)
ok: [192.168.0.59] => (item=readline)
ok: [192.168.0.59] => (item=openssl)
changed: [192.168.0.59] => (item=pv)
changed: [192.168.0.59] => (item=speedtest-cli)
ok: [192.168.0.59] => (item=sqlite)
changed: [192.168.0.59] => (item=thefuck)
changed: [192.168.0.59] => (item=tldr)
changed: [192.168.0.59] => (item=wget)
changed: [192.168.0.59] => (item=wrk)
changed: [192.168.0.59] => (item=zsh-autosuggestions)
changed: [192.168.0.59] => (item=zsh-autocomplete)

PLAY RECAP *****************************************************************************************************************************************************************************************************************
192.168.0.59               : ok=22   changed=9    unreachable=0    failed=0    skipped=35   rescued=0    ignored=0
```