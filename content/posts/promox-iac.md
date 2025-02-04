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

### Create user and role for terraform
```
$ pveum role add tfprovisioner -privs "Datastore.AllocateSpace Datastore.Audit Pool.Allocate SDN.Use Sys.Audit Sys.Console Sys.Modify Sys.PowerMgmt VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Migrate VM.Monitor VM.PowerMgmt"

$ pveum user add tfuser@pve --password secure1234

$ pveum aclmod / -user tfuser@pve -role tfprovisioner
```

## Configuration files
Please change the input variables based on your setup. Some useful links to grab resources from.
- [TrueNAS ISO image](https://download.sys.truenas.net/TrueNAS-SCALE-ElectricEel/24.10.2/TrueNAS-SCALE-24.10.2.iso)
- [Terraform Proxmox Provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)

### main.tf
```
resource "proxmox_vm_qemu" "truenas" {
  name        = "TrueNAS-scale"
  bios        = "seabios"
  cores       = 4
  memory      = 8192
  sockets     = 2
  scsihw      = "virtio-scsi-pci"
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
    id     = 0
    bridge = "vmbr0"
    model  = "virtio"
  }
}
```

### provider.tf
```
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "3.0.1-rc6"
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
promox_api_url = "https://192.168.0.230:8006/api2/json"
proxmox_node   = "pve"
```

## terraform init
```declarative
$ terraform init
Initializing the backend...
Initializing provider plugins...
- Finding telmate/proxmox versions matching "3.0.1-rc6"...
- Installing telmate/proxmox v3.0.1-rc6...
- Installed telmate/proxmox v3.0.1-rc6 (self-signed, key ID A9EBBE091B35AFCE)
```

## terraform apply
```
$ terraform apply

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # proxmox_vm_qemu.truenas will be created
  + resource "proxmox_vm_qemu" "truenas" {
      + additional_wait        = 5
      + agent                  = 0
      + automatic_reboot       = true
      + balloon                = 0
      + bios                   = "seabios"
      + boot                   = (known after apply)
      + bootdisk               = (known after apply)
      + ciupgrade              = false
      + clone_wait             = 10
      + cores                  = 4
      + cpu_type               = "host"
      + default_ipv4_address   = (known after apply)
      + default_ipv6_address   = (known after apply)
      + define_connection_info = true
      + desc                   = "Managed by Terraform."
      + force_create           = false
      + full_clone             = true
      + hotplug                = "network,disk,usb"
      + id                     = (known after apply)
      + kvm                    = true
      + linked_vmid            = (known after apply)
      + memory                 = 8192
      + name                   = "TrueNAS-scale"
      + onboot                 = false
      + protection             = false
      + reboot_required        = (known after apply)
      + scsihw                 = "virtio-scsi-pci"
      + skip_ipv4              = false
      + skip_ipv6              = false
      + sockets                = 2
      + ssh_host               = (known after apply)
      + ssh_port               = (known after apply)
      + tablet                 = true
      + tags                   = (known after apply)
      + target_node            = "pve"
      + unused_disk            = (known after apply)
      + vcpus                  = 0
      + vm_state               = "running"
      + vmid                   = 100

      + disks {
          + ide {
              + ide0 {
                  + cdrom {
                      + iso = "local:iso/TrueNAS-SCALE-24.10.2.iso"
                    }
                }
            }
          + scsi {
              + scsi0 {
                  + disk {
                      + backup               = true
                      + format               = "raw"
                      + id                   = (known after apply)
                      + iops_r_burst         = 0
                      + iops_r_burst_length  = 0
                      + iops_r_concurrent    = 0
                      + iops_wr_burst        = 0
                      + iops_wr_burst_length = 0
                      + iops_wr_concurrent   = 0
                      + linked_disk_id       = (known after apply)
                      + mbps_r_burst         = 0
                      + mbps_r_concurrent    = 0
                      + mbps_wr_burst        = 0
                      + mbps_wr_concurrent   = 0
                      + size                 = "50G"
                      + storage              = "local-lvm"
                    }
                }
            }
        }

      + network {
          + bridge    = "vmbr0"
          + firewall  = false
          + id        = 0
          + link_down = false
          + macaddr   = (known after apply)
          + model     = "virtio"
        }

      + smbios (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

proxmox_vm_qemu.truenas: Creating...
proxmox_vm_qemu.truenas: Still creating... [10s elapsed]
proxmox_vm_qemu.truenas: Creation complete after 11s [id=pve/qemu/100]

Stack trace from the terraform-provider-proxmox_v2.9.14 plugin:
```