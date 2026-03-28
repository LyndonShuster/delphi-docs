# delphi-docs

Replication guide and architectural reference for the Delphi homelab intelligence stack.

## What Is Delphi
Delphi is a Proxmox VE hypervisor running five LXC containers that form a
self-hosted, air-gap capable local intelligence infrastructure. It provides
private Git hosting, network-wide DNS filtering, mesh VPN, and a local RAG
inference stack that reasons over operator-authored documents entirely on-premises.

## Project Goals
- Demonstrate a production-grade self-hosted AI stack on modest hardware
- Provide a replicable deployment template for personal and client use
- Build toward a managed on-prem AI infrastructure offering for small businesses
  in resource-sector industries where data sovereignty is non-negotiable

## Who This Is For
- The operator rebuilding this stack on new or replacement hardware
- Anyone deploying a similar stack for a client site
- Anyone evaluating self-hosted RAG infrastructure

## Navigation

    architecture.md             system diagram, container roles, query flow

    host/
        proxmox-setup.md        install, repos, storage, backup, templates
        network.md              Tailscale, Pi-hole, DNS aliases

    containers/
        ct101-pihole.md         DNS filtering, ad blocking, Tailscale endpoint
        ct102-forgejo.md        private Git server, SSH, push-to-create
        ct103-ollama.md         inference engine, modelfiles, model zoo
        ct104-chromadb.md       vector store, semantic retrieval
        ct105-agent.md          LangGraph orchestration, FastAPI, chat dashboard

    operations/
        fresh-install.md        complete ordered deployment procedure
        hardware-sizing.md      VRAM table, RAM allocation, model selection

## Runtime Quick Reference

Start all containers:
    pct start 101 && pct start 102 && pct start 103 && pct start 104 && pct start 105

Check status:
    pct list

Access the agent dashboard:
    http://agent.home:8080

Trigger a manual corpus ingest:
    pct exec 105 -- /opt/agent-venv/bin/python3 /opt/agent/jobs/ingest.py corpus-general

Run a manual journal entry:
    pct exec 105 -- /opt/agent-venv/bin/python3 /opt/agent/jobs/journal.py

Check ingest log:
    pct exec 105 -- tail -50 /var/log/agent-ingest.log

## Related Repos

    git.home/Lyndon/delphi-agent      deployable agent application (CT 105)
    git.home/Lyndon/corpus-general    RAG corpus, ingested nightly at 3am

## Stack Services

    Service     URL                          Container
    Proxmox     https://HOST_IP:8006         host
    Pi-hole     http://pihole.home/admin     CT 101
    Forgejo     http://git.home:3000         CT 102
    Ollama      http://ollama.home:11434     CT 103
    ChromaDB    http://chroma.home:8000      CT 104
    Agent       http://agent.home:8080       CT 105
