# Homelab GP77: Passthrough de GPU y laboratorio RAG para CySA+

- **Fecha:** 2026-11-09
- **Entorno:** MSI Vector GP77 13VG (i7-13700H, 32 GB RAM, RTX 4070 Laptop de 8 GB), Proxmox VE 9.2, Ubuntu 26.04.1, Docker y Tailscale.
- **Estado:** Funcionando. Passthrough de GPU probado entre dos VM; RAG e inferencia mediante GPU verificados.

## Objetivo

Quería montar un laboratorio de IA separado para estudiar CySA+ sin perder mi VM de ComfyUI. La idea era que ambas máquinas virtuales pudieran utilizar la RTX 4070 del portátil, aunque nunca al mismo tiempo.

## Qué hice

1. **Creé la VM 305 (`ai-rag`)** con 8 vCPU, 12 GiB de RAM y un disco virtual de 120 GiB. Mantuve intacta la VM 300 (`comfyui`).
2. **Configuré el passthrough VFIO de la misma RTX 4070** (funciones PCI de vídeo y audio) para ambas VM. En la VM 305 instalé Ubuntu Server 26.04.1, el controlador NVIDIA 595-open y NVIDIA Container Toolkit.
3. **Desplegué Ollama, Open WebUI y Qdrant con Docker Compose.** Ollama ejecuta localmente `qwen3-embedding:4b`; DeepSeek Flash se encarga del chat mediante su API.
4. **Configuré el RAG de `CySA-Lab`** con búsqueda híbrida y un reranker MiniLM local. Subí un documento de prueba, consulté sus datos con citas y comprobé que los vectores se almacenaron en Qdrant.
5. **Corregí un problema de arranque de Docker.** Open WebUI intentaba enlazarse a la IP de Tailscale antes de que estuviera disponible (salida 128). Cambié el enlace a `127.0.0.1:3000` y publiqué el servicio de forma privada por HTTPS mediante Tailscale Serve.
6. **Transferí la GPU entre las VM (`305 → 300 → 305`)** apagándolas ordenadamente. Ambas volvieron a detectar la GPU sin reiniciar Proxmox.

## Resultados

- Qwen3 Embedding 4B mostró **100 % de uso de GPU**, con unos **3,4 GiB de VRAM** ocupados.
- La primera solicitud de embeddings tardó **34,8 s**, incluyendo el arranque del modelo; cinco solicitudes posteriores promediaron **20,1 ms**. Una prueba anterior en CPU tardó 2,2 s, aunque no eran mediciones directamente comparables.
- Los servicios Docker y Tailscale Serve volvieron a funcionar después de reiniciar la VM.
- También encontré un problema de compatibilidad con `reasoning_content` de DeepSeek al configurar el modo de razonamiento y las herramientas en mis modelos de estudio y cuestionarios.

## Aprendizajes y próximos pasos

La GPU no está compartida simultáneamente: apago una VM antes de iniciar la otra. Como el cambio manual funciona, decidí no automatizarlo. También comprobé que RAG no equivale a entrenar un modelo: los embeddings se calculan localmente y se guardan en Qdrant, mientras DeepSeek responde con el contexto recuperado.

Lo siguiente es preparar respaldos de las VM y desarrollar una herramienta de chat que procese y guarde tanto transcripciones como apuntes normales en `CySA-Lab`. **Esa herramienta de ingesta todavía no está implementada.**
