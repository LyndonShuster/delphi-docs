# Network

## Overview
Two network layers operate simultaneously.
LAN provides local connectivity between host and containers.
Tailscale provides encrypted remote access over the internet.
Pi-hole serves DNS for both layers.

## LAN
Assign static IPs to all containers at creation time via pct create.
No DHCP involvement. IPs do not change on reboot.

Recommended IP convention:
    HOST_IP.248     Proxmox host
    HOST_IP.53      CT 101 Pi-hole
    HOST_IP.54      CT 102 Forgejo
    HOST_IP.55      CT 103 Ollama
    HOST_IP.56      CT 104 ChromaDB
    HOST_IP.57      CT 105 Agent

The .53 assignment for Pi-hole is intentional — easy to remember,
consistent with the DNS port convention.

## Tailscale
Install on the Proxmox host:
    curl -fsSL https://tailscale.com/install.sh | sh

Enable IP forwarding before bringing Tailscale up:
    echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
    echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
    sysctl -p /etc/sysctl.d/99-tailscale.conf

Bring up as exit node, do not accept DNS on the host:
    tailscale up --advertise-exit-node --accept-dns=false

The --accept-dns=false flag is required. Pi-hole manages DNS for
the stack. Tailscale must not override it on the host.

Approve the exit node in the Tailscale admin console:
    https://login.tailscale.com/admin/machines
    Find your node → Edit Route Settings → approve exit node

## Tailscale in LXC Containers
Pi-hole requires its own Tailscale IP to serve as the tailnet nameserver.
Other containers do not need Tailscale installed.

Add TUN device passthrough to CT 101 config on the host before starting:
    echo 'lxc.cgroup2.devices.allow = c 10:200 rwm' >> /etc/pve/lxc/101.conf
    echo 'lxc.mount.entry = /dev/net/tun dev/net/tun none bind,create=file' \
    >> /etc/pve/lxc/101.conf

Restart CT 101 after adding these lines.
Install Tailscale inside CT 101 the same way as the host.
Use --accept-dns=false inside the container as well.

## Pi-hole DNS
Point your router's DHCP DNS setting to CT 101's LAN IP.
All LAN devices will then use Pi-hole for DNS resolution.

For Tailscale DNS:
    https://login.tailscale.com/admin/dns
    Add nameserver → Custom → CT 101 Tailscale IP
    Enable: Override local DNS
    Enable: Use with exit node

The Use with exit node setting is critical. Without it devices
using the exit node bypass Pi-hole entirely.

## DNS Aliases
Configure local DNS records in Pi-hole:
    Pi-hole admin → Local DNS → DNS Records

Required records for this stack:
    pihole.home     CT 101 LAN IP
    git.home        CT 102 LAN IP
    ollama.home     CT 103 LAN IP
    chroma.home     CT 104 LAN IP
    agent.home      CT 105 LAN IP

These aliases are used throughout the stack configuration.
All containers resolve each other by hostname not by IP.

## Pi-hole Listening Mode
Pi-hole must listen on all interfaces to answer Tailscale queries.
Default mode only listens on the LAN interface.

Set after install:
    pihole-FTL --config dns.listeningMode all
    systemctl restart pihole-FTL

## Air-Gap Behavior
Without internet:
    LAN DNS resolution continues normally
    All container services remain accessible on LAN
    Tailscale direct connections between LAN nodes continue
    Remote Tailscale access from outside the LAN requires internet

No stack functionality depends on internet connectivity
except remote access from outside the local network.