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
- [terraform](https://developer.hashicorp.com/terraform/install)

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
$ pveum role add tfprovisioner -privs "Datastore.AllocateSpace Datastore.Audit Pool.Allocate SDN.Use Sys.Audit Sys.Console Sys.Modify Sys.PowerMgmt VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor VM.PowerMgmt"

$ pveum user add tfuser@pve --password secure1234

$ pveum aclmod / -user tfuser@pve -role tfprovisioner
```

## Configuration files
Please change the input variables based on your setup. Some useful links to grab resources from.
- [Debian Cloud Images](https://cloud.debian.org/images/cloud/)
- [Terraform Proxmox Provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)

### main.tf
```
resource "proxmox_vm_qemu" "vm-instance" {
  name        = "vm-instance"
  target_node = "pve"
  clone       = var.cloudinit_template_name
  full_clone  = true
  cores       = 2
  memory      = 2048

  disk {
    size    = "32G"
    type    = "scsi"
    storage = "local-lvm"
    discard = "on"
  }

  network {
    model     = "virtio"
    bridge    = "vmbr0"
    firewall  = false
    link_down = false
  }
}
```

### provider.tf
```
terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
    }
  }
}

provider "proxmox" {
  pm_api_url      = var.promox_api_url
  pm_tls_insecure = true
}
```

### variables.tf
```
variable "cloudinit_template_name" {
  description = "The name of the Cloudinit template"
  type        = string
}

variable "promox_api_url" {
  description = "The Proxmox API URL"
  type        = string
}

variable "proxmox_node" {
  description = "The name of your Proxmox node"
  type        = string
}

variable "ssh_key" {
  description = "The SSH key to add to the virtual machine you're about to provision"
  sensitive   = true
  type        = string
}
```

### terraform.tfvars
```
pm_api_url = "https://192.168.0.200:8006/api2/json"
cloudinit_template_name = "debian-11-cloudinit-template"
proxmox_node = "pve"
ssh_key = "[YOUR_SSH_KEY]"
```

## terraform init
```declarative
$ terraform init
Initializing the backend...
Initializing provider plugins...
- Finding latest version of telmate/proxmox...
- Installing telmate/proxmox v2.9.14...
- Installed telmate/proxmox v2.9.14 (self-signed, key ID A9EBBE091B35AFCE)
```

## terraform plan
```declarative
$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
+ create

Terraform will perform the following actions:

  # proxmox_vm_qemu.vm-instance will be created
  + resource "proxmox_vm_qemu" "vm-instance" {
      + additional_wait           = 5
      + automatic_reboot          = true
      + balloon                   = 0
      + bios                      = "seabios"
      + boot                      = (known after apply)
      + bootdisk                  = (known after apply)
      + clone                     = "debian-11-cloudinit-template"
      + clone_wait                = 10
      + cores                     = 2
      + cpu                       = "host"
      + default_ipv4_address      = (known after apply)
      + define_connection_info    = true
      + force_create              = false
      + full_clone                = true
      + guest_agent_ready_timeout = 100
      + hotplug                   = "network,disk,usb"
      + id                        = (known after apply)
      + kvm                       = true
      + memory                    = 2048
      + name                      = "vm-instance"
      + nameserver                = (known after apply)
      + onboot                    = false
      + oncreate                  = true
      + preprovision              = true
      + reboot_required           = (known after apply)
      + scsihw                    = "lsi"
      + searchdomain              = (known after apply)
      + sockets                   = 1
      + ssh_host                  = (known after apply)
      + ssh_port                  = (known after apply)
      + tablet                    = true
      + target_node               = "pve"
      + unused_disk               = (known after apply)
      + vcpus                     = 0
      + vlan                      = -1
      + vmid                      = (known after apply)

      + disk {
          + backup             = true
          + cache              = "none"
          + discard            = "on"
          + file               = (known after apply)
          + format             = (known after apply)
          + iops               = 0
          + iops_max           = 0
          + iops_max_length    = 0
          + iops_rd            = 0
          + iops_rd_max        = 0
          + iops_rd_max_length = 0
          + iops_wr            = 0
          + iops_wr_max        = 0
          + iops_wr_max_length = 0
          + iothread           = 0
          + mbps               = 0
          + mbps_rd            = 0
          + mbps_rd_max        = 0
          + mbps_wr            = 0
          + mbps_wr_max        = 0
          + media              = (known after apply)
          + replicate          = 0
          + size               = "32G"
          + slot               = (known after apply)
          + ssd                = 0
          + storage            = "local-lvm"
          + storage_type       = (known after apply)
          + type               = "scsi"
          + volume             = (known after apply)
        }

      + network {
          + bridge    = "vmbr0"
          + firewall  = false
          + link_down = false
          + macaddr   = (known after apply)
          + model     = "virtio"
          + queues    = (known after apply)
          + rate      = (known after apply)
          + tag       = -1
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### terraform apply
It threw me an error but I can see the resource was provisioned.

```
...
Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

proxmox_vm_qemu.vm-instance: Creating...
proxmox_vm_qemu.vm-instance: Still creating... [10s elapsed]
proxmox_vm_qemu.vm-instance: Still creating... [20s elapsed]
╷
│ Error: Request cancelled
│ 
│   with proxmox_vm_qemu.vm-instance,
│   on main.tf line 2, in resource "proxmox_vm_qemu" "vm-instance":
│    2: resource "proxmox_vm_qemu" "vm-instance" {
│ 
│ The plugin.(*GRPCProvider).ApplyResourceChange request was cancelled.
╵

Stack trace from the terraform-provider-proxmox_v2.9.14 plugin:
```