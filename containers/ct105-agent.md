# CT 105 - Agent

## Purpose
Orchestration layer for the Delphi intelligence stack. Runs the LangGraph
query pipeline, FastAPI HTTP endpoint, corpus ingest pipeline, and journal
job. The only container that communicates with all other containers.

The application code for this container lives in a separate repo:
    git.home/Lyndon/delphi-agent

Deploy from that repo rather than configuring manually.

## Container Spec
    VMID:         105
    Hostname:     agent
    Cores:        2
    RAM:          2048MB
    Swap:         512MB
    Disk:         15GB
    IP:           192.168.x.57/24
    Unprivileged: yes
    Features:     nesting=1

## Installation

Enter the container:
    pct enter 105

Update and install dependencies:
    apt update && apt upgrade -y
    apt install -y python3 python3-pip python3-venv git sqlite3 curl

## DNS Resolution
CT 105 must resolve .home aliases to reach other containers.
Verify /etc/resolv.conf points to Pi-hole:
    cat /etc/resolv.conf

Should show:
    nameserver 192.168.x.53

If not, set it:
    echo "nameserver 192.168.x.53" > /etc/resolv.conf

## SSH Key for Forgejo
CT 105 needs git access to pull corpus repos and push journal entries.

Generate a dedicated key:
    ssh-keygen -t ed25519 -C "agent@delphi" -f /root/.ssh/agent_key -N ""

Configure SSH to use it for git.home:
    cat > /root/.ssh/config << EOF
    Host git.home
        HostName git.home
        User git
        IdentityFile /root/.ssh/agent_key
        IdentitiesOnly yes
    EOF

    chmod 600 /root/.ssh/config

Add the public key to Forgejo:
    cat /root/.ssh/agent_key.pub

Forgejo → Settings → SSH/GPG Keys → Add Key → paste the output.

Test authentication:
    ssh -T git@git.home

Should return a successful authentication message.

## Deploy Application

Clone the agent repo:
    git clone git@git.home:Lyndon/delphi-agent.git /opt/agent

Create the Python venv and install dependencies:
    python3 -m venv /opt/agent-venv
    source /opt/agent-venv/bin/activate
    pip install -r /opt/agent/requirements.txt

Initialize the SQLite database:
    cd /opt/agent
    python3 -c "from tools.sqlite import init_db; init_db()"

Create the repos directory for corpus clones:
    mkdir -p /opt/agent/repos

Clone designated corpus repos:
    cd /opt/agent/repos
    git clone git@git.home:Lyndon/corpus-general.git

Run initial ingest:
    source /opt/agent-venv/bin/activate
    python3 /opt/agent/jobs/ingest.py corpus-general --force

## Systemd Service

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

## Cron Jobs

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

Journal runs at 2am. Ingest runs at 3am so it picks up the journal entry.

## Global Git Config
Set the agent identity for all git operations:
    git config --global user.name "Agent Delphi"
    git config --global user.email "agent@delphi.home"
    git config --global init.defaultBranch main

## Verify
    systemctl status agent
    curl http://localhost:8080/health

Health endpoint should return model name and collection list.
Access the chat dashboard at http://agent.home:8080
API documentation at http://agent.home:8080/docs