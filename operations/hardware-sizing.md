# Hardware Sizing

## Host Requirements

Minimum:
    CPU:     4 cores
    RAM:     8GB
    Storage: 100GB SSD
    OS:      x86_64

Recommended:
    CPU:     8+ cores
    RAM:     16GB+
    Storage: 500GB+ SSD
    GPU:     8GB+ VRAM for accelerated inference

The stack runs on modest hardware. The i7-3770 from 2012 with 16GB RAM
is the reference deployment. Inference is slow but functional.
GPU acceleration dramatically improves inference speed but is not required.

## Container RAM Allocation

    CT 101  Pi-hole      512MB    fixed, does not scale with host RAM
    CT 102  Forgejo      1024MB   fixed, scales slightly with repo count
    CT 103  Ollama       see table below
    CT 104  ChromaDB     2048MB   fixed, scales with corpus size
    CT 105  Agent        2048MB   fixed

Reserve at least 25% of total host RAM for the Proxmox host itself.

On a 16GB host:
    Containers use ~12GB total
    Host retains ~4GB
    This is the minimum comfortable allocation

On a 32GB host:
    CT 103 can be increased to 16GB
    Enables comfortable 14B model inference

## CT 103 RAM by Model Size

    Model size    Minimum CT 103 RAM    Comfortable CT 103 RAM
    0.5B          2GB                   4GB
    1.5B          3GB                   4GB
    3B            4GB                   6GB
    7B            6GB                   8GB
    14B           10GB                  14GB
    32B           20GB                  24GB

Rule of thumb: allocate at least 1.5x the model file size in RAM.
Add 1GB overhead for the Ollama server process itself.

If a model fails to load with an insufficient memory error despite
adequate RAM allocation, run from the Proxmox host:
    echo 3 > /proc/sys/vm/drop_caches

Linux buff/cache can appear as unavailable memory to processes
inside unprivileged LXC containers.

## Model Selection by VRAM

The modelfile FROM line is the only change needed when swapping models.
All other modelfile content and config.py model names stay the same.

    VRAM      Recommended base model     Notes
    2GB       qwen2.5:1.5b               functional, slow
    4GB       qwen2.5:3b                 good balance, reference deployment
    8GB       qwen2.5:7b                 noticeably smarter
    12GB      qwen2.5:14b                strong reasoning capability
    24GB      qwen2.5:32b                near GPT-4 class on focused tasks

For delphi-analyst (reasoning model):
    VRAM      Recommended base model
    8GB       deepseek-r1:7b             reference deployment
    12GB      deepseek-r1:14b            significantly better reasoning
    24GB      deepseek-r1:32b            strong analytical capability

nomic-embed-text is required regardless of hardware and uses
approximately 275MB RAM. It is CPU-only and does not benefit from GPU.

## GPU Passthrough
GPU passthrough to a VM or container enables hardware-accelerated inference.
Requirements:
    IOMMU enabled in BIOS (VT-d for Intel, AMD-Vi for AMD)
    GPU in its own IOMMU group or ACS override patch applied
    Host GPU driver blacklisted, vfio-pci bound instead

GPU passthrough significantly reduces inference time:
    CPU only (i7-3770):   3B model ~2-4 minutes per complex query
    GPU (8GB VRAM):       3B model ~5-10 seconds per complex query

For GPU deployments increase CT 103 disk allocation to 40GB+
as larger models require more storage.

## Storage Allocation

    CT 101  4GB     Pi-hole logs and database
    CT 102  30GB    Git repositories, scales with repo count
    CT 103  20GB    Model storage, increase for larger models
    CT 104  10GB    ChromaDB vectors, scales with corpus size
    CT 105  15GB    Application code, corpus clones, SQLite state

CT 103 storage by model:
    nomic-embed-text    275MB
    qwen2.5:0.5b        400MB
    qwen2.5:1.5b        1GB
    qwen2.5:3b          2GB
    qwen2.5:7b          4.7GB
    qwen2.5:14b         9GB
    deepseek-r1:7b      4.7GB

CT 104 storage scales with corpus. 10GB handles thousands of markdown
documents comfortably. Increase if ingesting large binary or PDF corpora.

## Scaling Up
The stack is designed to scale without architectural changes.

Swap a model: edit the FROM line in the relevant modelfile on CT 103,
recreate the named model, restart CT 105. No other changes required.

Add RAM to CT 103: from the Proxmox host:
    pct set 103 --memory NEW_RAM_MB
    pct stop 103 && pct start 103

Add a corpus repo: clone into /opt/agent/repos/ on CT 105,
add to the cron line in /etc/cron.d/agent-jobs, run initial ingest.

Add an edge node: any device on the Tailscale tailnet with git
installed can push data to Forgejo. CT 105 ingests it automatically.
