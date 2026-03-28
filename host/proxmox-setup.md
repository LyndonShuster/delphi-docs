# Proxmox Setup

## Installation
Download Proxmox VE ISO from https://www.proxmox.com/downloads
Write to USB and boot the target machine through the installer.
Assign a static IP during install — this becomes the permanent management address.
Access the web UI after install at https://HOST_IP:8006

## Repository Configuration
The default install includes enterprise repos requiring a paid subscription.
Disable them before any package operations or apt will return 401 errors.

Via GUI:
    Node → Repositories → select each enterprise repo → Disable

Repos to disable:
    enterprise.proxmox.com/debian/pve
    enterprise.proxmox.com/debian/ceph-squid

Add the free repo via GUI:
    Node → Repositories → Add → No-Subscription

Then update:
    apt update && apt upgrade -y

## Known Package Issue
If pct create fails after a fresh install, check for libbinutils corruption:
    apt reinstall libbinutils

This affects container creation and must be resolved before deploying any containers.

## Storage
A single disk install creates two pools automatically:
    local       ISO templates, backups (directory)
    local-lvm   Container disk images (LVM-thin)

No additional configuration required for single disk deployments.

## SSD Health
fstrim.timer runs weekly by default. Verify it is active:
    systemctl status fstrim.timer

Run manually on first install:
    fstrim -av

First run on a fresh install commonly reclaims significant space.
Enable the timer if not already active:
    systemctl enable --now fstrim.timer

## Backup Job
    Datacenter → Backup → Add

    Node:        your Proxmox hostname
    Storage:     local
    Schedule:    Sun 02:00
    Selection:   all containers
    Mode:        Snapshot
    Compression: ZSTD
    Max Backups: 3

## Container Templates
Download the Debian 12 template before creating containers:
    pveam update
    pveam download local debian-12-standard_12.12-1_amd64.tar.zst

Verify:
    pveam list local

Debian 13 templates cause LXC hook failures on PVE 9.1. Use Debian 12.

## Container Creation
All containers in this stack use the same base command pattern:

    pct create VMID local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
      --hostname NAME \
      --cores CORES \
      --memory RAM_MB \
      --swap SWAP_MB \
      --rootfs local-lvm:DISK_GB \
      --net0 name=eth0,bridge=vmbr0,ip=192.168.x.IP/24,gw=192.168.x.1 \
      --unprivileged 1 \
      --features nesting=1 \
      --start 1

See operations/fresh-install.md for exact values per container.
See operations/hardware-sizing.md for resource allocation guidance.

## Node Naming
The default hostname is pve. Renaming via hostnamectl alone corrupts
the pmxcfs cluster database and breaks the web UI.

Recommendation: accept pve as the OS hostname and use Pi-hole DNS
aliases for human-readable naming. The Tailscale machine name can
be set independently without affecting Proxmox.

If a rename is required, stop all containers first, follow the
official Proxmox node migration procedure, and verify pve-cluster
is healthy before restarting services.