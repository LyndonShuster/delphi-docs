# Fresh Install

Complete ordered procedure for deploying the Delphi stack on a new machine.
Follow in sequence. Each step assumes the previous is complete.

## 1. Proxmox Host

Install Proxmox VE on target hardware.
Set static IP during install.
Access web UI at https://HOST_IP:8006

Disable enterprise repos and add no-subscription repo:
    Node → Repositories → disable both enterprise repos
    Node → Repositories → Add → No-Subscription
    apt update && apt upgrade -y

Fix libbinutils if needed:
    apt reinstall libbinutils

Run fstrim:
    fstrim -av
    systemctl enable --now fstrim.timer

Download Debian 12 container template:
    pveam update
    pveam download local debian-12-standard_12.12-1_amd64.tar.zst

Configure backup job:
    Datacenter → Backup → Add
    Mode: Snapshot, Compression: ZSTD, Max Backups: 3

## 2. Tailscale on Host

    curl -fsSL https://tailscale.com/install.sh | sh

    echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
    echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
    sysctl -p /etc/sysctl.d/99-tailscale.conf

    tailscale up --advertise-exit-node --accept-dns=false

Authenticate via the URL printed in the terminal.
Approve exit node in Tailscale admin console.

## 3. Create Containers

Replace 192.168.x with your subnet. Replace .1 with your gateway last octet.

CT 101 Pi-hole:
    pct create 101 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
      --hostname pihole --cores 1 --memory 512 --swap 256 \
      --rootfs local-lvm:4 \
      --net0 name=eth0,bridge=vmbr0,ip=192.168.x.53/24,gw=192.168.x.1 \
      --unprivileged 1 --features nesting=1 --start 1

CT 102 Forgejo:
    pct create 102 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
      --hostname forgejo --cores 2 --memory 1024 --swap 512 \
      --rootfs local-lvm:30 \
      --net0 name=eth0,bridge=vmbr0,ip=192.168.x.54/24,gw=192.168.x.1 \
      --unprivileged 1 --features nesting=1 --start 1

CT 103 Ollama:
    pct create 103 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
      --hostname ollama --cores 4 --memory 8192 --swap 2048 \
      --rootfs local-lvm:20 \
      --net0 name=eth0,bridge=vmbr0,ip=192.168.x.55/24,gw=192.168.x.1 \
      --unprivileged 1 --features nesting=1 --start 1

CT 104 ChromaDB:
    pct create 104 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
      --hostname chromadb --cores 2 --memory 2048 --swap 512 \
      --rootfs local-lvm:10 \
      --net0 name=eth0,bridge=vmbr0,ip=192.168.x.56/24,gw=192.168.x.1 \
      --unprivileged 1 --features nesting=1 --start 1

CT 105 Agent:
    pct create 105 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
      --hostname agent --cores 2 --memory 2048 --swap 512 \
      --rootfs local-lvm:15 \
      --net0 name=eth0,bridge=vmbr0,ip=192.168.x.57/24,gw=192.168.x.1 \
      --unprivileged 1 --features nesting=1 --start 1

Verify all running:
    pct list

## 4. CT 101 - Pi-hole

Add TUN passthrough from host before entering container:
    echo 'lxc.cgroup2.devices.allow = c 10:200 rwm' >> /etc/pve/lxc/101.conf
    echo 'lxc.mount.entry = /dev/net/tun dev/net/tun none bind,create=file' \
    >> /etc/pve/lxc/101.conf
    pct stop 101 && pct start 101

    pct enter 101
    apt update && apt upgrade -y && apt install -y curl

Pre-seed config and install:
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

    curl -sSL https://install.pi-hole.net | bash
    pihole-FTL --config webserver.api.password YOURPASSWORD
    pihole-FTL --config dns.listeningMode all
    systemctl restart pihole-FTL

Install Tailscale:
    curl -fsSL https://tailscale.com/install.sh | sh
    systemctl enable --now tailscaled
    tailscale up --accept-dns=false

Note the Pi-hole Tailscale IP:
    tailscale ip -4

    exit

## 5. Configure Pi-hole DNS

Add local DNS records at http://pihole.home/admin:
    Local DNS → DNS Records

    pihole.home     192.168.x.53
    git.home        192.168.x.54
    ollama.home     192.168.x.55
    chroma.home     192.168.x.56
    agent.home      192.168.x.57

Configure Tailscale DNS at https://login.tailscale.com/admin/dns:
    Add nameserver → Custom → Pi-hole Tailscale IP
    Enable: Override local DNS
    Enable: Use with exit node

## 6. CT 102 - Forgejo

    pct enter 102
    apt update && apt upgrade -y && apt install -y git curl wget

    adduser --system --shell /bin/bash --gecos 'Git Version Control' \
      --group --disabled-password --home /home/git git

    wget -O /usr/local/bin/forgejo \
      https://codeberg.org/forgejo/forgejo/releases/download/vX.Y.Z/forgejo-X.Y.Z-linux-amd64
    chmod 755 /usr/local/bin/forgejo

    mkdir -p /var/lib/forgejo/{custom,data,log} /etc/forgejo
    chown -R git:git /var/lib/forgejo /etc/forgejo
    chmod 750 /etc/forgejo

    cat > /etc/systemd/system/forgejo.service << EOF
    [Unit]
    Description=Forgejo
    After=network.target
    [Service]
    RestartSec=2s
    Type=simple
    User=git
    Group=git
    WorkingDirectory=/var/lib/forgejo
    ExecStart=/usr/local/bin/forgejo web --config /etc/forgejo/app.ini
    Restart=always
    Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/forgejo
    [Install]
    WantedBy=multi-user.target
    EOF

    systemctl daemon-reload
    systemctl enable --now forgejo

Complete first-run installer at http://git.home:3000
Use SQLite, set server domain to git.home, create admin account.

Enable push-to-create:
    nano /etc/forgejo/app.ini

Add under [repository]:
    ENABLE_PUSH_CREATE_USER = true
    DEFAULT_PUSH_CREATE_PRIVATE = true

    systemctl restart forgejo
    exit

Create required repos in Forgejo (all private):
    corpus-general
    delphi-agent
    delphi-docs

## 7. CT 103 - Ollama

    pct enter 103
    apt update && apt upgrade -y && apt install -y curl zstd
    curl -fsSL https://ollama.com/install.sh | sh

Add to [Service] in /etc/systemd/system/ollama.service:
    Environment="OLLAMA_HOST=0.0.0.0"
    Environment="OLLAMA_KEEP_ALIVE=10m"
    Environment="OLLAMA_MAX_LOADED_MODELS=1"

    systemctl daemon-reload && systemctl restart ollama

Pull models:
    ollama pull nomic-embed-text
    ollama pull qwen2.5:0.5b
    ollama pull qwen2.5:3b
    ollama pull deepseek-r1:7b

Create modelfile directory and build named models:
    mkdir -p /modelfiles

See containers/ct103-ollama.md for modelfile contents.

    ollama create delphi-rag -f /modelfiles/delphi-rag.modelfile
    ollama create delphi-extractor -f /modelfiles/delphi-extractor.modelfile
    ollama create delphi-analyst -f /modelfiles/delphi-analyst.modelfile
    ollama list
    exit

## 8. CT 104 - ChromaDB

    pct enter 104
    apt update && apt upgrade -y
    apt install -y python3 python3-pip python3-venv curl

    python3 -m venv /opt/chromadb-venv
    source /opt/chromadb-venv/bin/activate
    pip install chromadb

    mkdir -p /var/lib/chromadb

    cat > /opt/start-chromadb.sh << EOF
    #!/bin/bash
    source /opt/chromadb-venv/bin/activate
    chroma run --host 0.0.0.0 --port 8000 --path /var/lib/chromadb
    EOF

    chmod +x /opt/start-chromadb.sh

    cat > /etc/systemd/system/chromadb.service << EOF
    [Unit]
    Description=ChromaDB Vector Store
    After=network.target
    [Service]
    Type=simple
    User=root
    ExecStart=/opt/start-chromadb.sh
    Restart=always
    RestartSec=3
    [Install]
    WantedBy=multi-user.target
    EOF

    systemctl daemon-reload
    systemctl enable --now chromadb
    curl http://localhost:8000/api/v2/heartbeat
    exit

## 9. CT 105 - Agent

    pct enter 105
    apt update && apt upgrade -y
    apt install -y python3 python3-pip python3-venv git sqlite3 curl

Set DNS:
    echo "nameserver 192.168.x.53" > /etc/resolv.conf

Set git identity:
    git config --global user.name "Agent Delphi"
    git config --global user.email "agent@delphi.home"
    git config --global init.defaultBranch main

Generate SSH key and add to Forgejo:
    ssh-keygen -t ed25519 -C "agent@delphi" -f /root/.ssh/agent_key -N ""

    cat > /root/.ssh/config << EOF
    Host git.home
        HostName git.home
        User git
        IdentityFile /root/.ssh/agent_key
        IdentitiesOnly yes
    EOF

    chmod 600 /root/.ssh/config
    cat /root/.ssh/agent_key.pub

Add the public key to Forgejo → Settings → SSH/GPG Keys.

Test:
    ssh -T git@git.home

Deploy application:
    git clone git@git.home:Lyndon/delphi-agent.git /opt/agent
    python3 -m venv /opt/agent-venv
    source /opt/agent-venv/bin/activate
    pip install -r /opt/agent/requirements.txt

    cd /opt/agent
    python3 -c "from tools.sqlite import init_db; init_db()"
    mkdir -p repos

Clone corpus and run initial ingest:
    cd /opt/agent/repos
    git clone git@git.home:Lyndon/corpus-general.git
    cd /opt/agent
    python3 jobs/ingest.py corpus-general --force

Install systemd service:
    cat > /etc/systemd/system/agent.service << EOF
    [Unit]
    Description=Delphi Agent FastAPI
    After=network.target
    [Service]
    Type=simple
    User=root
    WorkingDirectory=/opt/agent
    ExecStart=/opt/agent-venv/bin/uvicorn main:app --host 0.0.0.0 --port 8080
    Restart=always
    RestartSec=3
    [Install]
    WantedBy=multi-user.target
    EOF

    systemctl daemon-reload
    systemctl enable --now agent

Configure cron:
    cat > /etc/cron.d/agent-jobs << EOF
    PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
    HOME=/root
    0 2 * * * root GIT_SSH_COMMAND="ssh -i /root/.ssh/agent_key -o StrictHostKeyChecking=no" \
    /opt/agent-venv/bin/python3 /opt/agent/jobs/journal.py \
    >> /var/log/agent-journal.log 2>&1
    0 3 * * * root GIT_SSH_COMMAND="ssh -i /root/.ssh/agent_key -o StrictHostKeyChecking=no" \
    /opt/agent-venv/bin/python3 /opt/agent/jobs/ingest.py corpus-general \
    >> /var/log/agent-ingest.log 2>&1
    EOF

    exit

## 10. Verify Full Stack

From the Proxmox host:
    pct list

All five containers should show running.

    tailscale status

Delphi and Pi-hole nodes should show connected.

From a browser:
    http://agent.home:8080/health

Should return model name, collections, and available models.

    http://agent.home:8080

Chat dashboard should load and respond to queries.

## 11. Full Reboot Test

    reboot

After the host comes back:
    pct list
    pct exec 101 -- systemctl status pihole-FTL
    pct exec 101 -- tailscale status
    pct exec 103 -- systemctl status ollama
    pct exec 104 -- systemctl status chromadb
    pct exec 105 -- systemctl status agent

All services should be active without manual intervention.