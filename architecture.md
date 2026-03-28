# Architecture

## System Diagram

    Proxmox VE Host
    ├── CT 101  Pi-hole          192.168.x.53   pihole.home
    ├── CT 102  Forgejo          192.168.x.54   git.home:3000
    ├── CT 103  Ollama           192.168.x.55   ollama.home:11434
    ├── CT 104  ChromaDB         192.168.x.56   chroma.home:8000
    └── CT 105  Agent            192.168.x.57   agent.home:8080

## Container Roles

CT 101 Pi-hole: DNS resolver for LAN and Tailscale tailnet.
Blocks ad and tracker domains. Resolves .home aliases to container IPs.
Runs its own Tailscale node so it can serve as the tailnet nameserver.

CT 102 Forgejo: Self-hosted Git server. Source of truth for all
documents, code, and agent-generated content. The RAG corpus lives
here as versioned markdown files.

CT 103 Ollama: Local inference engine. Serves language models and
embedding models via HTTP API. Hosts custom modelfiles that tune
model behavior for specific roles in the agent pipeline.

CT 104 ChromaDB: Vector store. Holds document embeddings generated
by nomic-embed-text. Answers semantic similarity queries at inference
time. Knows nothing about models or documents directly.

CT 105 Agent: Orchestration layer. Runs the LangGraph query pipeline,
FastAPI HTTP endpoint, ingest pipeline, and journal job. The only
container that talks to all others.

## Two Logical Groups

Platform services — independent, always on:
    CT 101  Pi-hole
    CT 102  Forgejo

Agent cluster — tightly coupled, work as a unit:
    CT 103  Ollama     the brain
    CT 104  ChromaDB   the memory
    CT 105  Agent      the hands

Platform services have no dependency on the agent cluster.
The agent cluster depends on platform services for DNS and document storage.

## Query Flow

    POST /query → CT 105 FastAPI
                      ↓
              embed query via nomic (CT 103)
                      ↓
              semantic search ChromaDB (CT 104)
              returns top 5 relevant chunks
                      ↓
              fetch last 5 session turns from SQLite
                      ↓
              send chunks + query + history to Ollama (CT 103)
                      ↓
              return sourced response to caller
              log turn to SQLite

## Corpus Flow

    Documents committed to Forgejo repo
                ↓
    2am  journal.py detects changes, writes daily summary back to Forgejo
    3am  ingest.py pulls latest, embeds changed files only, updates ChromaDB
                ↓
    Agent reasons over updated corpus on next query

## Network Layer

Two network layers operate simultaneously:

LAN: standard subnet, all containers reachable by static IP or .home alias.
Tailscale: encrypted mesh overlay, all tailnet devices reach Delphi services
by Tailscale IP. Pi-hole serves as tailnet DNS so .home aliases resolve
for remote devices too.

Air-gap behavior: all LAN operations function without internet.
Tailscale remote access requires internet. Nothing else does.

## Design Principles

Single responsibility: each container has one job.
Resilience: every service restarts automatically, survives reboots.
Data sovereignty: nothing leaves the host network during inference.
Air-gap capable: full functionality without internet connectivity.
Hardware agnostic: modelfiles abstract model selection from application code.