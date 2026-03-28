# CT 103 - Ollama

## Purpose
Local inference engine and model zoo. Serves language models and
embedding models via HTTP API. Hosts custom modelfiles that define
named model roles used by the agent pipeline.

## Container Spec
    VMID:         103
    Hostname:     ollama
    Cores:        4
    RAM:          8192MB
    Swap:         2048MB
    Disk:         20GB
    IP:           192.168.x.55/24
    Unprivileged: yes
    Features:     nesting=1

RAM allocation must be at least 1.5x the size of the largest model
file intended to run. See operations/hardware-sizing.md for guidance.

## Installation

Enter the container:
    pct enter 103

Update and install dependencies:
    apt update && apt upgrade -y && apt install -y curl zstd

Install Ollama:
    curl -fsSL https://ollama.com/install.sh | sh

## Systemd Configuration
Ollama binds to localhost by default. Add environment variables
to make it network accessible and tune memory behavior:

    nano /etc/systemd/system/ollama.service

Add under [Service]:
    Environment="OLLAMA_HOST=0.0.0.0"
    Environment="OLLAMA_KEEP_ALIVE=10m"
    Environment="OLLAMA_MAX_LOADED_MODELS=1"

    systemctl daemon-reload
    systemctl enable --now ollama

OLLAMA_HOST=0.0.0.0 makes Ollama reachable from other containers.
OLLAMA_KEEP_ALIVE=10m keeps the last used model warm between queries.
OLLAMA_MAX_LOADED_MODELS=1 prevents multiple models loading simultaneously.

## Memory Note
Buff/cache on the host can prevent Ollama from loading larger models
inside the container even when total RAM appears sufficient.
If a model fails to load with an insufficient memory error run from
the Proxmox host:
    echo 3 > /proc/sys/vm/drop_caches

Then retry the model load.

## Pull Base Models

    ollama pull nomic-embed-text
    ollama pull qwen2.5:0.5b
    ollama pull qwen2.5:3b
    ollama pull deepseek-r1:7b

Adjust model selection based on available VRAM.
See operations/hardware-sizing.md for the full model selection table.

nomic-embed-text is required. It handles all document and query
embedding for the RAG pipeline and must always be present.

## Modelfiles
Custom modelfiles live at /modelfiles/ inside this container.
Create the directory:
    mkdir -p /modelfiles

### delphi-rag
Default query model. Factual, citation-focused, low temperature.

    cat > /modelfiles/delphi-rag.modelfile << EOF
    FROM qwen2.5:3b
    PARAMETER temperature 0.3
    PARAMETER num_ctx 8192
    PARAMETER repeat_penalty 1.1
    PARAMETER top_p 0.9
    SYSTEM """
    You are a precise document retrieval assistant.
    Answer questions based ONLY on provided context documents.
    Always cite your sources by filename.
    If the context does not contain enough information say so clearly.
    Never speculate beyond what the documents contain.
    """
    EOF

    ollama create delphi-rag -f /modelfiles/delphi-rag.modelfile

### delphi-extractor
Fast structured extraction. Minimal temperature for deterministic output.

    cat > /modelfiles/delphi-extractor.modelfile << EOF
    FROM qwen2.5:0.5b
    PARAMETER temperature 0.1
    PARAMETER num_ctx 4096
    PARAMETER repeat_penalty 1.2
    SYSTEM """
    You are a structured data extraction assistant.
    Extract specific fields, dates, names, and values from documents.
    Return only what is explicitly present in the text.
    Format output as clean structured lists.
    Never infer or guess missing information.
    """
    EOF

    ollama create delphi-extractor -f /modelfiles/delphi-extractor.modelfile

### delphi-analyst
Deep reasoning model. Chain-of-thought, higher temperature.

    cat > /modelfiles/delphi-analyst.modelfile << EOF
    FROM deepseek-r1:7b
    PARAMETER temperature 0.5
    PARAMETER num_ctx 8192
    SYSTEM """
    You are an analytical reasoning assistant.
    Think carefully before answering complex questions.
    Reason across multiple documents to synthesize insights.
    Identify gaps, inconsistencies, and implications in provided context.
    Cite sources for every claim.
    """
    EOF

    ollama create delphi-analyst -f /modelfiles/delphi-analyst.modelfile

## Verify
    ollama list
    curl http://localhost:11434/api/tags

All four base models and three named modelfiles should appear in the list.
The API should be reachable from other containers at http://ollama.home:11434