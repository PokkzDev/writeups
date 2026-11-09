# GP77 Homelab: GPU Handoff and a CySA+ RAG Lab

- **Date:** 2026-11-09
- **Environment:** MSI Vector GP77 13VG (i7-13700H, 32 GB RAM, RTX 4070 Laptop 8 GB), Proxmox VE 9.2, Ubuntu 26.04.1, Docker, Tailscale.
- **Status:** Working. GPU passthrough tested between two VMs; RAG and GPU inference verified.

## Goal

I wanted a separate AI study lab for CySA+ without giving up my existing ComfyUI VM. Both VMs should use the laptop's RTX 4070, just never at the same time.

## What I did

1. **Created VM 305 (`ai-rag`)** with 8 vCPUs, 12 GiB RAM and a 120 GiB virtual disk. VM 300 (`comfyui`) stayed intact.
2. **Configured the same VFIO-passed RTX 4070** (GPU and audio PCI functions) on both VMs. I installed Ubuntu Server 26.04.1, NVIDIA driver 595-open and NVIDIA Container Toolkit in VM 305.
3. **Deployed Ollama, Open WebUI and Qdrant with Docker Compose.** Ollama runs local `qwen3-embedding:4b`; DeepSeek Flash provides the chat model via API.
4. **Set up `CySA-Lab` RAG** with hybrid search and a local MiniLM reranker. I uploaded a test document, queried its facts with citations and verified vectors were stored in Qdrant.
5. **Fixed a Docker startup problem.** Open WebUI was binding to my Tailscale IP before it was assigned (exit 128). I switched it to `127.0.0.1:3000` and exposed it privately over HTTPS with Tailscale Serve.
6. **Switched the GPU `305 → 300 → 305`** using graceful shutdowns. Both guests detected the GPU again without rebooting Proxmox.

## Results

- Qwen3 Embedding 4B reported **100% GPU offload**, occupying about **3.4 GiB VRAM**.
- First GPU embedding request: **34.8 s** including startup; five warm requests averaged **20.1 ms**. My earlier CPU request took 2.2 s, but the tests weren't directly comparable.
- Docker services and Tailscale Serve recovered after a VM reboot.
- I also encountered a DeepSeek thinking/tool-call `reasoning_content` compatibility issue while configuring study and quiz presets.

## Lessons / next steps

The GPU isn't shared simultaneously: I shut down one VM before starting the other. Manual handoff works well enough that I skipped automation. RAG also isn't training: embeddings are computed locally and stored in Qdrant, while DeepSeek answers using retrieved context.

Next, I want proper VM backups and a chat tool that can prepare and ingest either transcripts or ordinary notes into `CySA-Lab`. **That ingestion tool isn't implemented yet.**
