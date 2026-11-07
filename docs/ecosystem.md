# Ecosystem — runtime support, registries, RAG vs parametric, audience, base-model churn

Status: **pending** — research thread still running on 2026-11-07; this file will be replaced with the full findings. Interim facts from docs/mechanisms.md:

- Only LoRA adapters load in all three runtimes (llama.cpp, Ollama, vLLM) today; cartridges need a loader we write.
- vLLM: one LoRA per request, dynamic loading behind `VLLM_ALLOW_RUNTIME_LORA_UPDATING`; non-prefix KV reuse via LMCache CacheBlend.
- llama.cpp: multiple LoRAs, hot-swap via `POST /lora-adapters`; control vectors at launch; slot save/restore is prefix-only.
- Ollama: single `ADAPTER` per Modelfile.
