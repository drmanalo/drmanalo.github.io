+++
date = "2026-09-16T15:48:46+01:00"
draft = false
title = "Windows Setup"
tags = ["ansible","windows"]
+++

## Prerequisites
To safely run Linux commands inside a Windows machine, it is better to install Linux inside Windows. With Windows Subsystem for Linux (WSL), we can install Linux distribution and use Linux applications, utilities and Bash command-line tools directly on Windows. This removes the overhead of a traditional `virtual machine`.
- [Windows Subsytem](https://learn.microsoft.com/en-us/windows/wsl/install)

## Install Ubuntu
```
PS C:\Users\drman> wsl --install -d Ubuntu
Downloading: Ubuntu
Installing: Ubuntu                                          
Distribution successfully installed. It can be launched via 'wsl.exe -d Ubuntu'
Launching Ubuntu...
Provisioning the new WSL instance Ubuntu
This might take a while...
Create a default Unix user account:
```

<!--more-->

## Install Ansible on Ubuntu
```
PS C:\Users\drman> wsl
$ sudo apt update
$ sudo apt install -y pipx
$ pipx ensurepath
# restart the shell or: exec $SHELL

$ pipx install --include-deps ansible
```

## Setting mirrored networking mode
Add this to your `.wslconfig` in your Windows user profile, then restart WSL:
```ini
[wsl2]
networkingMode=mirrored
```

```
PS C:\Users\drman> wsl --shutdown
PS C:\Users\drman> wsl
```

You shoud be able to ping your Windows host machine.

```
$ ping 192.168.`50.52
PING 192.168.150.52 (192.168.150.52) 56(84) bytes of data.
64 bytes from 192.168.150.52: icmp_seq=1 ttl=64 time=0.039 ms
64 bytes from 192.168.150.52: icmp_seq=2 ttl=64 time=0.038 ms
64 bytes from 192.168.150.52: icmp_seq=3 ttl=64 time=0.056 ms
```

## Install OpenSSH Server on Windows
Install the optional feature and start the service:
```
PS C:\windows\system32> Add-WindowsCapability -Online -Name OpenSSH.Server*

Path          :
Online        : True
RestartNeeded : False

PS C:\windows\system32> Start-Service sshd
PS C:\windows\system32> Set-Service -Name sshd -StartupType Automatic
```

### Lock the firewall down to the WSL client

```
PS C:\windows\system32> Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP" | Select Name, Enabled

Name                    Enabled
----                    -------
OpenSSH-Server-In-TCP   True

PS C:\windows\system32> Set-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -RemoteAddress 127.0.0.1
```

By default, sshd drops you into `cmd.exe`. Set PowerShell as the default shell instead:
```
PS C:\windows\system32> New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```

Test the connection from Ubuntu. I preauthorised my public key to the Windows authorised keys on file `C:\ProgramData\ssh\administrators_authorized_keys`
```
$ ssh -i ~/.ssh/ansible_windows drman@127.0.0.1
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\drman> exit
Connection to 127.0.0.1 closed
```

## Sample ansible configuration
I wrote my initial Ansible configuration and tested it on my local machine.

```
$ ansible-playbook chocolatey.yml 

PLAY [Install applications via Chocolatey] *****************************************************

TASK [Gathering Facts] *************************************************************************
ok: [localhost]

TASK [Ensure Chocolatey is installed] **********************************************************
ok: [localhost]

TASK [Install applications] ********************************************************************
ok: [localhost] => (item=git)
changed: [localhost] => (item=golang)
ok: [localhost] => (item=googlechrome)
changed: [localhost] => (item=hugo)
changed: [localhost] => (item=winfetch)
ok: [localhost] => (item=vscode)

PLAY RECAP ************************************************************************************
localhost  : ok=3   changed=1   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```