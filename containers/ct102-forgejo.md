# CT 102 - Forgejo

## Purpose
Self-hosted Git server. Source of truth for all documents, code, and
agent-generated content. The RAG corpus and agent application code
live here as versioned repositories.

## Container Spec
    VMID:         102
    Hostname:     forgejo
    Cores:        2
    RAM:          1024MB
    Swap:         512MB
    Disk:         30GB
    IP:           192.168.x.54/24
    Unprivileged: yes
    Features:     nesting=1

## Installation

Enter the container:
    pct enter 102

Update and install dependencies:
    apt update && apt upgrade -y && apt install -y git curl wget

Create the git system user Forgejo runs as:
    adduser --system --shell /bin/bash --gecos 'Git Version Control' \
      --group --disabled-password --home /home/git git

Download the Forgejo binary:
    wget -O /usr/local/bin/forgejo \
      https://codeberg.org/forgejo/forgejo/releases/download/vX.Y.Z/forgejo-X.Y.Z-linux-amd64

Check https://forgejo.org/releases for the current stable version.
Replace X.Y.Z with the current release number.

    chmod 755 /usr/local/bin/forgejo

Create required directories:
    mkdir -p /var/lib/forgejo/{custom,data,log} /etc/forgejo
    chown -R git:git /var/lib/forgejo /etc/forgejo
    chmod 750 /etc/forgejo

## Systemd Service

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

## First Run
Open the web installer at http://git.home:3000

Recommended settings:
    Database:       SQLite3
    Server Domain:  git.home
    HTTP Port:      3000
    Base URL:       http://git.home:3000/

Create the admin account in the installer before clicking install.

## Push To Create
Enables creating repos via git push without pre-creating them in the UI.
Edit app.ini:
    nano /etc/forgejo/app.ini

Add or update under [repository]:
    [repository]
    ENABLE_PUSH_CREATE_USER = true
    DEFAULT_PUSH_CREATE_PRIVATE = true

Restart Forgejo:
    systemctl restart forgejo

## SSH Keys
Add SSH keys for any user or service that needs git access.
Forgejo web UI → Settings → SSH/GPG Keys → Add Key

Required keys for this stack:
    CT 105 agent key    allows agent to pull corpus repos and push journal entries
    Operator key        allows pushing from dev machines on the tailnet

## Repositories
Create these repos after install. All private:
    corpus-general      RAG corpus, ingested nightly by CT 105
    delphi-agent        CT 105 application code
    delphi-docs         this documentation

## Verify
    systemctl status forgejo

Access at http://git.home:3000