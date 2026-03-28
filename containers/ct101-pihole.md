# CT 101 - Pi-hole

## Purpose
DNS resolver and ad blocker for the local network and Tailscale tailnet.
Runs its own Tailscale node so it can serve as the global nameserver
for all tailnet devices, providing ad blocking and .home alias resolution
regardless of whether the querying device is on the LAN or remote.

## Container Spec
    VMID:       101
    Hostname:   pihole
    Cores:      1
    RAM:        512MB
    Swap:       256MB
    Disk:       4GB
    IP:         192.168.x.53/24
    Unprivileged: yes
    Features:   nesting=1

## TUN Device Passthrough
Required before starting the container. Run from the Proxmox host:

    echo 'lxc.cgroup2.devices.allow = c 10:200 rwm' >> /etc/pve/lxc/101.conf
    echo 'lxc.mount.entry = /dev/net/tun dev/net/tun none bind,create=file' \
    >> /etc/pve/lxc/101.conf

Without this Tailscale will fail to start inside the container.

## Installation

Enter the container:
    pct enter 101

Update and install curl:
    apt update && apt upgrade -y && apt install -y curl

Pre-seed the static IP config before running the installer.
The Pi-hole v6 installer exits if it cannot detect a static IP
configured in the traditional Linux network files:

    mkdir -p /etc/pihole
    cat > /etc/pihole/setupVars.conf << EOF
    PIHOLE_INTERFACE=eth0
    IPV4_ADDRESS=192.168.x.53/24
    IPV6_ADDRESS=
    PIHOLE_DNS_1=1.1.1.1
    PIHOLE_DNS_2=8.8.8.8
    QUERY_LOGGING=true
    INSTALL_WEB_SERVER=true
    INSTALL_WEB_INTERFACE=true
    LIGHTTPD_ENABLED=true
    CACHE_SIZE=10000
    DNS_FQDN_REQUIRED=true
    DNS_BOGUS_PRIV=true
    DNSMASQ_LISTENING=local
    WEBPASSWORD=
    EOF

Run the installer:
    curl -sSL https://install.pi-hole.net | bash

Set the admin password after install:
    pihole-FTL --config webserver.api.password YOURPASSWORD

## Listening Mode
Pi-hole must listen on all interfaces to answer Tailscale queries:
    pihole-FTL --config dns.listeningMode all
    systemctl restart pihole-FTL

Default mode is local which drops queries arriving on the Tailscale
tun interface. This must be set before configuring Tailscale DNS.

## Tailscale
Install Tailscale inside the container:
    curl -fsSL https://tailscale.com/install.sh | sh
    systemctl enable --now tailscaled
    tailscale up --accept-dns=false

Note the Tailscale IP assigned to this container:
    tailscale ip -4

This IP is used in the Tailscale admin console as the global nameserver.

## Local DNS Records
Add .home aliases after the stack is fully deployed:
    Pi-hole admin → Local DNS → DNS Records

See host/network.md for the full list of required records.

## Verify
    systemctl status pihole-FTL
    systemctl status tailscaled
    tailscale status

Access the admin panel at http://pihole.home/admin or http://192.168.x.53/admin