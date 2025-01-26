+++
date = "2025-01-26T20:53:18Z"
draft = false
title = "Promox Iac"
tags = ["linux","proxmox"]
+++

## Prerequisites
Being an `archlinux` user means dependencies are for `arch` but this blog post should work with any `nix` distribution.
- [Linux](https://wiki.archlinux.org/title/Installation_guide)
- [Proxmox](https://www.proxmox.com/en/products/proxmox-virtual-environment/get-started)

## Install procedure
I downloaded the latest Proxmox release using [this link](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso). Then I install `balena-etcher` to create the USB bootdisk. I then boot using the created bootdisk and followed the installation screen. It's vital that you configure the network correctly otherwise you will have an unreachable Proxmox instance.

![proxmox_network](../proxmox-network-config.png)

## Post install
```
$ bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/misc/post-pve-install.sh)"
 ✓ Corrected Proxmox VE Sources
 ✓ Disabled 'pve-enterprise' repository
 ✓ Enabled 'pve-no-subscription' repository
 ✓ Corrected 'ceph package repositories'
 ✓ Added 'pvetest' repository
 ✓ Disabled subscription nag (Delete browser cache)
 ✓ Disabled high availability
 ✓ Updated Proxmox VE
 ✓ Completed Post Install Routines
```

## Prepare for IaC
```
$ apt install libguestfs-tools

$ wget https://cloud.debian.org/images/cloud/bullseye/latest/debian-11-generic-amd64.qcow2

$ virt-customize -a debian-11-generic-amd64.qcow2 --install qemu-guest-agent
[   0.0] Examining the guest ...
[   7.2] Setting a random seed
virt-customize: warning: random seed could not be set for this type of guest
[   7.2] Setting the machine ID in /etc/machine-id
[   7.2] Installing packages: qemu-guest-agent
[  14.5] Finishing off
```

### Create VM template
```
$ qm create 9003 --name "debian-11-cloudinit-template" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0

$ qm importdisk 9003 debian-11-generic-amd64.qcow2 local-lvm
unused0: successfully imported disk 'local-lvm:vm-9003-disk-0'

$ qm set 9003 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9003-disk-0
update VM 9003: -scsi0 local-lvm:vm-9003-disk-0 -scsihw virtio-scsi-pci

$ qm set 9003 --boot c --bootdisk scsi0
update VM 9003: -boot c -bootdisk scsi0

$ qm set 9003 --ide2 local-lvm:cloudinit
update VM 9003: -ide2 local-lvm:cloudinit
  Logical volume "vm-9003-cloudinit" created.
ide2: successfully created disk 'local-lvm:vm-9003-cloudinit,media=cdrom'
generating cloud-init ISO

$ qm set 9003 --serial0 socket --vga serial0
update VM 9003: -serial0 socket -vga serial0

$ qm set 9003 --agent enabled=1
update VM 9003: -agent enabled=1

$ qm template 9003
  Renamed "vm-9003-disk-0" to "base-9003-disk-0" in volume group "pve"
  Logical volume pve/base-9003-disk-0 changed.
  WARNING: Combining activation change with other commands is not advised.
```

### Create user and role for terraform
```
$ pveum role add tfrole -privs "Datastore.AllocateSpace Datastore.Audit Pool.Allocate Sys.Audit Sys.Console Sys.Modify VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor VM.PowerMgmt"

$ pveum user add tfuser@pve --password secure1234

$ pveum aclmod / -user tfuser@pve -role tfrole
```

## Configuration files
Please change the input variables based on your setup.

### main.tf
```

variable "cloudinit_template_name" {
    type = string 
}

variable "proxmox_node" {
    type = string
}

variable "ssh_key" {
  type = string 
  sensitive = true
}

resource "proxmox_vm_qemu" "k8s-1" {
  count = 3
  name = "k8s-1${count.index + 1}"
  target_node = var.proxmox_node
  clone = var.cloudinit_template_name
  agent = 1
  os_type = "cloud-init"
  cores = 4
  sockets = 1
  cpu = "host"
  memory = 4096
  scsihw = "virtio-scsi-pci"
  bootdisk = "scsi0"

  disk {
    slot = 0
    size = "40G"
    type = "scsi"
    storage = "pve1"
  }

  network {
    model = "virtio"
    bridge = "vmbr2"
  }
  
  lifecycle {
    ignore_changes = [
      network,
    ]
  }

  ipconfig0 = "ip=172.20.0.20${count.index + 1}/24,gw=172.20.0.1"
  nameserver = "172.20.0.31"
  
  sshkeys = <<EOF
  ${var.ssh_key}
  EOF
}
```

### provider.tf
```
variable "pm_api_url" {
  type = string
}

terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
      version = "2.9.11"
    }
  }
}

provider "proxmox" {
  pm_api_url = var.pm_api_url
}
```

### terraform.tfvars
```
pm_api_url = "https://192.168.0.200:8006/api2/json"
cloudinit_template_name = "debian-11-cloudinit-template"
proxmox_node = "dc"
ssh_key = "[YOUR_SSH_KEY]"
```
