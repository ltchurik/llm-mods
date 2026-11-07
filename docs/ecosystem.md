# Ecosystem — runtimes, registries, RAG vs parametric, audience, base-model churn

Researched 2026-11-07. Runtime API details are consolidated from docs/mechanisms.md (sources there) plus the searches cited below.

## A. Runtime support today

| Runtime | LoRA | Multiple LoRAs at once | Hot-load | Control vectors | Precomputed / trained KV prefix | Notes |
|---|---|---|---|---|---|---|
| **llama.cpp / llama-server** | yes (`--lora`, GGUF adapter) | yes, scaled | yes, `POST /lora-adapters` | yes, `--control-vector[-scaled]`, layer range; launch-time only (hot-swap = open issue #10685) | slot save/restore to `.bin` (`--slot-save-path`); **prefix-only reuse**, no trained-KV load | the engine under Ollama, LM Studio, Jan, KoboldCpp, GPT4All |
| **Ollama** | one `ADAPTER` per Modelfile; safetensors (Llama/Mistral/Gemma arches) or GGUF; must match exact base | no | no (rebuild model) | no | no | 8.9 M developers claimed; registry of 4,500+ models |
| **LM Studio** | GGUF/MLX models; adapter support limited to what llama.cpp/MLX expose in the UI | — | — | — | — | closed source; HF browser built in |
| **vLLM** | PEFT LoRA, `--lora-modules`, dynamic `/v1/load_lora_adapter` behind `VLLM_ALLOW_RUNTIME_LORA_UPDATING` | **one per request** | yes | no (EasySteer needs a vLLM fork; EasyEdit2 partial via worker hooks) | prefix caching; **non-prefix** KV reuse via LMCache CacheBlend connector (config only) | the multi-user runtime |
| **SGLang** | multi-LoRA serving | per request | yes | no | RadixAttention prefix cache; used by Cartridges authors only for synthesis/throughput | — |
| **Tokasaurus** (`geoff/cartridges` branch) | — | — | — | — | **yes — the only runtime that loads trained cartridges** (`/v1/cartridge/chat/completions`, list of cartridges per request) | Llama 3 / Qwen2 arches; from source; tested on A100 80 GB |
| **HF transformers + PEFT** | yes | yes (`add_weighted_adapter`, `set_adapter([...])`) | yes | via libraries | prefix-tuning params supported | reference implementation path, not a serving stack |
| **LoRAX** (Predibase, Apache-2.0) | thousands per base; HF Hub / S3 / local | continuous multi-adapter batching; "blend multiple adapters per request" on roadmap | yes, just-in-time | no | no | one base model per deployment; targets q/k/v/o |

Sources: llama.cpp server README and PR #5970; Ollama Modelfile docs; vLLM LoRA docs; LMCache blending docs; Tokasaurus cartridges branch; LoRAX README (https://github.com/predibase/lorax).

**Bottom line.** A LoRA is the only payload every consumer runtime loads today. Trained-KV cartridges load nowhere mainstream. Control vectors load only in llama.cpp. Knowledge edits must be re-merged into base weights (new GGUF). KV Packet / CacheBlend-style concatenation exists only in HF-transformers code or vLLM-via-LMCache.

## B. Things that resemble a mod store or adapter registry

| thing | what it is | versioning / compatibility | quality verification |
|---|---|---|---|
| **Hugging Face Hub (PEFT adapters)** | 135,000+ GGUF models, uncounted PEFT adapters; `adapter_config.json` names `base_model_name_or_path` | free-text base name, no hash binding, no runtime matrix | none (model cards, downloads) |
| **Ollama library** | 4,500+ models, pull by name/tag | tags per quant; adapters only via user Modelfiles | none |
| **LoRAX / Predibase** | serving + adapter loading from Hub/S3 | adapter must match deployed base (user's job) | none |
| **Civitai** (image LoRAs — the cultural analogue) | marketplace of LoRAs/checkpoints for SD1.5/SDXL/Flux/Hunyuan etc. | every listing carries a **"Base Model" field**; search filters by base; LoRAs do not survive base upgrades ("upgrade fatigue"), research like LoRA-X (arXiv 2501.16559) tries to port adapters across versions but is limited to similar architectures | community ratings/images; no automated eval |
| **Parametric RAG** (Su et al., arXiv 2501.15915) | research: parameterise each document into LoRA matrices merged into FFN; "Retrieve-Update-Generate"; up to 36% lower inference time than in-context RAG on HotpotQA/2Wiki/CWQ | per base model | benchmark only |
| **Claude/Codex "mods" marketplaces, llm-wiki, skills repos** | prompts, skills, hooks for agents | n/a — text, not weights | none |
| **Model-merging ecosystem** (mergekit, MergeForge, Ballast) | merge/version checkpoints and deltas | Ballast: content-addressed deltas, merges as views | none automated |

**Nobody** binds an artefact to a base **hash + tokenizer hash**, publishes **before/after + retention evals**, declares a **memory budget**, or records **composition rules**. Civitai proves the catalog model works for a hobbyist community once a "Base Model" field exists; it also shows the churn cost.

## C. RAG vs parametric — what the literature says

**For parametric (cartridge/adapter) over RAG**
- Latency: fine-tuned/parametric answers skip the retrieval hop; typical published figures put RAG at ~200–800 ms vs 50–200 ms without retrieval (codesota guide; several 2026 decision guides agree on the direction, numbers vary).
- Cross-document reasoning: Active Reading (Meta, arXiv 2508.09494) argues parametric knowledge lets the model relate pieces "where simple retrieval augmentation falls short", and notes RAG pipelines are "considerably more complex, with larger storage cost, higher memory and compute requirements … and higher latency".
- Parametric RAG: up to 36% lower inference time while matching or exceeding in-context RAG baselines (survey arXiv 2503.10677 §7.5).
- Offline and simplicity: a file vs an embedding model + vector store + retriever + chunking policy.
- Version pinning: LibEvoBench / evolveQA show models are "version-oblivious"; docs in context help +10–20 pts (see docs/corpora.md) — a pinned-version mod gives that without per-query retrieval.

**For RAG over parametric**
- Freshness: re-indexing is minutes; any parametric artefact is frozen until rebuilt.
- Oracle-retrieval RAG still beats closed-book: Active Reading reports "a substantial gap between the RAG setting and what can be achieved in the closed-book setting" even after their method.
- Unsupervised fine-tuning learns new facts poorly (Ovadia et al. 2024, cited in arXiv 2502.04095): "RAG shows superior performance in knowledge-intensive tasks compared to unsupervised fine-tuning". Cartridges/D2L use *distillation* rather than next-token fine-tuning precisely to avoid this, but no paper compares them to RAG head-to-head.
- Off-context damage: Cartridges++ shows cartridges degrade unrelated behaviour; RAG leaves the model untouched.
- Scale: RAG indexes billions of chunks; a cartridge covers one corpus of ~100k–500k tokens.

**The honest reading**: the 2026 consensus is "RAG for knowledge, fine-tune for behaviour, hybrid in production". Mods win only in the niche the kill test targets — pinned, offline, cross-document, composable — and must be measured there, with mod+RAG as a fourth arm.

## D. Audience size (local-model users)

| signal | figure | source |
|---|---|---|
| Ollama GitHub stars | ~175–185k (2026 reports range 162k–185k) | aiproductivity.ai, maketocreate.com |
| Ollama users | "nearly 9 M users", $65 M raise (TechCrunch, 2026-07-09); 8.9 M developers on ollama.com | Wikipedia/TechCrunch; tech-insider.org |
| Ollama downloads | 52 M/month in Q1 2026; 2.5 B lifetime model downloads | falcao.org |
| llama.cpp stars | >73k | maketocreate.com |
| Open WebUI stars | ~143k | maketocreate.com |
| GGUF models on HF | 135,000+ | falcao.org |
| r/LocalLLaMA | >250k members | toolchase.com |
| LM Studio | closed source; ~19k stars for its SDK repos; v0.4.28 (Oct 2026) | presenc.ai, aiproductivity.ai |

Implication: the installer audience is millions of Ollama/llama.cpp users, but the *mod-aware* slice (people who would install knowledge rather than paste docs) is unknown; r/LocalLLaMA (~250k) is the realistic launch surface.

## E. Base-model churn (≤ ~9B open releases, 2026)

| date | release | sizes ≤ 9B | licence |
|---|---|---|---|
| Jan 2026 | Phi-4 Mini | 3.8B | MIT |
| Feb 2026 | Gemma 3 refresh | 1B/4B | Apache-2.0 |
| Feb 2026 | Mistral Small 3.2 | (24B) | Apache-2.0 |
| 2026-02-16 | **Qwen3.5** | 0.8B, 2B, 4B, 9B | Apache-2.0 |
| Mar 2026 | Mistral Small 4 | (119B MoE) | Apache-2.0 |
| 2026-04-02 | **Gemma 4** | E2B, E4B (12B in June) | Apache-2.0 |
| 2026-04-16 | **Qwen3.6** | small sizes continue | Apache-2.0 |
| Aug 2026 | Qwen3.8 27B / Flash-Next | (27B) — new hybrid attention, Qwen4 preview | Apache-2.0 |
| 2026 | Granite 4.x | 3B, 8B | Apache-2.0 |
| Dec 2025 | Ministral 3 | 3B, 8B, 14B | Apache-2.0 |
| ongoing | SmolLM3 3B, Llama 3.2 1B/3B | — | various |

Sources: computingforgeeks.com comparison, hidekazu-konishi.com timeline, promptquorum.com, aicassindra.com SLM landscape, vdf.ai.

**Rate**: the two families with the strongest small-model ecosystems (Qwen, Gemma) each shipped a new generation every ~2 months in H1 2026, and Qwen3.8 changed the attention architecture. A cartridge is bound to one checkpoint; a mod catalog therefore needs a per-base rebuild pipeline and should expect **~4–6 rebuilds per year per family**. Note also that Doc-to-LoRA hypernets exist for gemma-2-2b, Mistral-7B-v0.2 and Qwen3-4B — already one or two generations behind.

## Implications for a mod loader

1. v0 must ship its own loader for trained-KV cartridges; the cleanest paths are (a) Tokasaurus fork, (b) a vLLM plugin that injects a KV prefix (CacheBlend/LMCache shows the connector route works without patching vLLM core), (c) a llama.cpp patch extending slot restore to accept a trained cache. Pick (b) or (c) by where the audience is: (c) reaches Ollama/LM Studio users eventually, (b) reaches servers now.
2. Keep the mod format runtime-neutral (payload + manifest); runtimes become backends. If a runtime later adds native cartridge loading, that is a win, not a threat.
3. LoRA-type mods (Text-to-LoRA skills, trained deltas) work everywhere today — ship them first for reach, cartridges for the knowledge story.
4. Manifest must bind to base **sha256 + tokenizer sha256**, not a model name — Hugging Face and Ollama both show how name-only binding fails.
5. Per-base catalog from day one; budget ~5 rebuilds/year/family; automate `modctl make` so a rebuild is a cron job on the Spark, not a project.
6. Evaluate against **four arms**: plain, RAG, mod, mod+RAG. Publish the kill test either way.
7. Adopt Cartridges++ data mixing by default and record `budget` (KV positions) in every manifest; retention eval (IFEval + general QA) is mandatory for the verified badge.
8. Launch on r/LocalLLaMA + Show HN with the version-pinning benchmark ("does your local model know Postgres 18?"), where the literature already shows models fail.
9. Civitai's "Base Model" field is the minimum viable compatibility UX; we add hash binding, budget and eval badges on top.
10. Private team stores are a static index + object storage; defer any backend until there are external makers.
