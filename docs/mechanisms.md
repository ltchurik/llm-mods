# Mechanisms — what a "no-training-for-the-user" mod can be made of

Researched 2026-11-07. Sources fetched directly (arXiv, GitHub, HF). "Not stated" = the source gives no number.

## Summary table

| Mechanism | What a mod contains | Make cost | Size | Composable | Base-model-bound | Quality caveat | Runtime support | Code + licence |
|---|---|---|---|---|---|---|---|---|
| **Cartridges** | Trained KV prefix (`.pt`, K/V per layer) | ~30 min on 8×H100 for Llama-8B, 100k–484k-token corpus; needs a synthesis server | 52 MB–0.96 GB (LongHealth); 470 MB for 2048-tok/Llama-3B fp32 | Yes, concatenate at inference; tested pairwise | Yes (exact model) | Off-context derailment: MMLU/IFEval drop, context contamination; fix = Cartridges++ data mixing or router | Tokasaurus branch only; no vLLM/llama.cpp/Ollama loader | Apache-2.0, HazyResearch/cartridges; Cartridges++ no code |
| **Doc-to-LoRA** | Generated rank-8 LoRA per 8K chunk (in-process) | Per doc <1 s; hypernetwork per base ~5 days 8×H200 | Not stated; <100 MB inference overhead | Within-doc rank concat; cross-doc stacking untested | Yes (per-base hypernet; gemma-2-2b, Mistral-7B, Qwen3-4B released) | 82.5% of ICL on SQuAD; interference on unrelated queries; Mistral-7B weak | HF-transformers code; LoRA export not documented | MIT, SakanaAI/doc-to-lora |
| **Text-to-LoRA** | PEFT LoRA r=8 q/v from a task description | ~5 days 1×H100 per base hypernet; generation instant | Small | Not evaluated | Yes (Llama-3.1-8B, gemma-2-2b, Mistral-7B) | Behaviour/task, not knowledge | Standard LoRA → llama.cpp, Ollama, vLLM | Apache-2.0, SakanaAI/text-to-lora |
| **Knowledge editing** (MEMIT/AlphaEdit/UltraEdit) | Modified MLP `down_proj` matrices (ship merged weights or tensor diff) | MEMIT 10k edits ≈0.9 h (+6.5 h precompute) on GPT-J; UltraEdit 20k edits <5 min on a 7B/24 GB GPU | Tensor diff: tens–hundreds MB per layer | Edits accumulate; degrades ~1–3k (MEMIT), ~5–8k (AlphaEdit) sequential | Yes; fails on Phi-3-3.8B, Gemma2-2B | General capability collapse at scale (MMLU 0.61→0.11 at 10k) | Re-merged weights only | MIT (EasyEdit, MEMIT, AlphaEdit); UltraEdit no licence file |
| **KV Packet** | Precomputed doc KV + 16 global soft tokens | Wrapper once per model (30 epochs, 1×A100); per doc one prefill | KV of doc + few-KB adapter | Yes — designed for concatenating independent docs | Yes | HotpotQA/MusiQue F1 0.42–0.43; distribution-sensitive | HF transformers only | MIT, ChuangtaoChen-TUM/KVPacket |
| **Steering / control vectors** | One `n_embd` vector per layer (GGUF `direction.N` / `.pt`) | ~60 s on a 4090 for 7B | Sub-MB | Yes, summed with scales | Yes (model + layer) | Behaviour only; garbles at high strength; no MoE in repeng | llama.cpp/llama-server native (launch-time); vLLM needs fork/hooks; Ollama none | repeng MIT; EasyEdit2 MIT; EasySteer Apache-2.0 |
| **LatentSkill / skill→LoRA** | Generated rank-8 LoRA on all 7 projections per layer | Compiler 2×10 epochs on 8×H100 per base; per skill one forward pass | Not stated | Weighted sum; component-aligned only; α>1.2 collapses | Yes (Qwen3-8B; Llama3-8B check) | Tested on ALFWorld/Search-QA only | Output is LoRA → in principle any runtime; export not documented | Apache-2.0, yuaofan0-oss/LatentSkill |
| **Weight-space programming** (MIT) | Exact tensor diff `edit.npz` + manifest (131,072 weights) | GPT-6 Astra, 8.5–20 h wall-clock, no gradients | ~0.26 MB diff | Not studied | Yes (Qwen3.5-0.8B exact revision) | n=1; Gemma 270M no gain | Apply via script → full safetensors → any runtime | Logs CC BY 4.0, scripts MIT; harness not released |

## 1. Cartridges (Hazy Research, ICLR 2026) and Cartridges++

**What it is.** A trained KV cache (prefix-tuning parameterisation) for a specific corpus, trained offline via "self-study" (synthetic self-quiz conversations + context distillation), loaded at inference instead of putting the corpus in context. https://arxiv.org/abs/2506.06266

**Size vs corpus.** Sizes tried: 128–8192 tokens; corpora 100k–484k tokens. LongHealth (~100k tokens) → 0.96 GB cartridge (13.8× smaller than ICL KV) scoring 55.1% (above full ICL), or 52 MB (256× compression) scoring 47.7% with 121× throughput. MTOB 484k-token textbook → 0.54 GB for Llama-3.1-8B, 36.1 chrF (Llama 4 Scout 109B with full textbook: 36.3). https://hazyresearch.stanford.edu/blog/2025-06-08-cartridges — Released artifact `hazyresearch/cartridge-wauoq23f`: 470 MB, Llama-3.2-3B-Instruct, 2048 tokens, fp32 (bf16 export would be ~235 MB). https://huggingface.co/hazyresearch/cartridge-wauoq23f

**Training cost.** "An ICL-quality Cartridge for Llama-8B takes about 30 minutes on one 8×H100 node (unoptimized)"; batch 64, seq 1024, 1 epoch over tens of thousands of synthetic samples. Repo defaults: lr 2e-2, 1 epoch, global batch 32; synthesis needs a Tokasaurus or SGLang server. https://arxiv.org/html/2506.06266v3 · https://github.com/HazyResearch/cartridges

**Quality.** Matches ICL on average; beats ICL by 11.0 chrF on MTOB; prefix-tuning beats LoRA parameterisation by 4.5 chrF and keeps MMLU (54.7→54.3) where LoRA drops to 45.3. **No RAG/BM25 baseline in the paper.** https://arxiv.org/html/2506.06266v3

**Composition.** Four ~100k-token 10-K cartridges composed pairwise by concatenating caches, no joint training; composed > single > truncated ICL. Tokasaurus accepts a `cartridges` list per request. Limit: only same-model, same-type corpora, pairwise. https://github.com/ScalingIntelligence/tokasaurus/tree/geoff/cartridges

**Off-context derailment (Cartridges++, arXiv 2609.35621, 2026-09-28).** Cartridges degrade *off-context* behaviour more than the full context does: fabricated/misattributed quotes on unrelated queries, drops on TinyMMLU/MMLU-Pro/IFEval. Fixes: (a) data mixing — 1–10% of self-study pairs replaced with off-document instruct queries distilled from the no-cartridge model (γ=0.05 recommended, no inference cost); (b) kNN router (k=10, α=0.05) deciding whether to prefill with the cartridge; >90% accept on-context / reject off-context. Longer training does not fix it. Cartridges also "often answer as if absent evidence were present". Models: Qwen3-4B, Qwen3-4B-Instruct-2507, Gemma-4-E4B-it. **No code.** https://arxiv.org/html/2609.35621v1

**Base models in code.** Qwen3-4B (custom `FlexQwen3ForCausalLM`), Llama-3.2-3B-Instruct (custom class). **Licence** Apache-2.0. **Serving** Tokasaurus branch `geoff/cartridges` only (`/v1/cartridge/chat/completions`); vLLM not integrated.

**Fast alternative.** Attention Matching (arXiv 2602.16284): up to 50× KV compaction in seconds; sits between Cartridges and eviction on fidelity/preservation.

## 2. Doc-to-LoRA and Text-to-LoRA (Sakana AI)

**D2L.** Perceiver-style hypernetwork (309M params) emits a rank-8 LoRA on every MLP down-projection per 8K-token chunk in one forward pass; chunk adapters concatenate along rank. https://arxiv.org/html/2602.15902v1
- Released hypernets: `gemma_2b_d2l` (gemma-2-2b-it) 1.26 GB, `mistral_7b_d2l` 2.30 GB, `qwen_4b_d2l` (Qwen3-4B-Instruct-2507) 1.68 GB. https://huggingface.co/SakanaAI/doc-to-lora
- Meta-training: ~5 days on 8×H200, ~900M tokens; **one hypernetwork per target LLM.**
- Quality: SQuAD ROUGE-L 0.814 = 82.5% of ICL upper bound; update latency 0.21–0.55 s vs ~40 s oracle context distillation; **no RAG comparison**; Mistral-7B near no-context baseline; interference on replaced contexts.
- Max length: NIAH near-perfect to ~40K tokens; QA tested to 32K.
- Stacking LoRAs from different documents **not evaluated**. API: `model.internalize(doc)`, `generate()`, `reset()`. LoRA export not documented.
- Licence: MIT (code). https://github.com/SakanaAI/doc-to-lora

**T2L.** Released for Llama-3.1-8B-Instruct, gemma-2-2b-it, Mistral-7B-Instruct-v0.2; ~5 days on one H100 per base. Emits a **standard PEFT LoRA** (r=8, q/v) → loads in any LoRA runtime. Avg benchmark: Llama-8B 73.0→77.2. Behaviour, not knowledge. Apache-2.0. https://github.com/SakanaAI/text-to-lora

## 3. Knowledge editing

- **MEMIT**: GPT-J 10k CounterFact edits: efficacy 98.9 / generalisation 88.6 / specificity 73.7; 10k edits ≈ 0.9 h + 6.5 h precompute. MIT. https://arxiv.org/pdf/2210.07229
- **Sequential collapse**: on LLaMA3-8B with 100-edit rounds, MEMIT holds to ~1,000 edits and near-collapses by ~2,000; AlphaEdit fully collapses by ~8,000. https://arxiv.org/pdf/2601.11042
- **AlphaEdit**: null-space projection, +36.7% over baselines; reproducibility study: works on Llama3.2-1B/3B, Qwen2.5-3B; **fails on Phi3-3.8B and Gemma2-2B**; degradation from ~5k edits; Llama3-8B MMLU 0.61→0.11 at 10k. MIT. https://arxiv.org/pdf/2606.26783
- **UltraEdit**: up to 2M edits, 20K edits on Mistral-7B in <5 min on a 24 GB GPU; edits layers 29–30 `down_proj`; no licence file. https://github.com/XiaojieGu/UltraEdit
- **EasyEdit**: MIT toolkit; `sequential_edit=True`; only GRACE supports rollback; LLaMA-2-7B editing needs 40 GB+. https://github.com/zjunlp/EasyEdit
- **Portable delta**: no framework defines one; a mod would ship the modified `down_proj` tensors (or a diff) and re-merge before loading. No runtime loads it natively.

## 4. KV Packet and concatenable KV

- **KV Packet** (arXiv 2604.13226): [8 header soft tokens][doc][8 trailer soft tokens], KV precomputed; keys RoPE-rotated at serve time; caches concatenated with zero recompute. Global wrapper trained once per model (fp32, a few KB). F1 on Llama: Biography 0.95, HotpotQA 0.42, MusiQue 0.43, NIAH 0.71; TTFT 1.36–19.45× lower. Assumes independent documents. MIT, HF-transformers backend. https://github.com/ChuangtaoChen-TUM/KVPacket
- **LMCache CacheBlend (vLLM)**: non-prefix KV reuse with partial recompute (~15%); config-only on vLLM via the LMCache connector. https://docs.lmcache.ai/kv_cache_optimizations/blending.html
- **llama.cpp slot save/restore**: `--slot-save-path` + `POST /slots/{id}?action=save|restore`; prefix-only reuse, not concatenation. https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md

## 5. Steering / control vectors

- **repeng** (MIT): per-layer control vector from contrasting pairs in <1 min; exports GGUF for llama.cpp; no MoE. https://github.com/vgel/repeng
- **llama.cpp**: `--control-vector`, `--control-vector-scaled`, `--control-vector-layer-range`; multiple summed; launch-time only (hot-swap is open issue #10685). https://github.com/ggerganov/llama.cpp/pull/5970
- **EasyEdit2** (MIT): CAA, LM-Steer, SAE steering, etc.; partial vLLM via worker hooks. **EasySteer** (Apache-2.0): vLLM-based but needs a vLLM fork.
- Limits: behaviour/style/safety dials, not facts; model-and-layer specific.

## 6. Textual skills → LoRA

- **LatentSkill** (arXiv 2606.06087): hypernetwork "skill compiler" → rank-8 LoRA on all projections of frozen Qwen3-8B; composition by weighted sum (component-level merging 84.6% vs direct 69.2%); α>1.2 collapses. Code Apache-2.0, checkpoints on HF. https://github.com/yuaofan0-oss/LatentSkill
- **Parametric Skills** (arXiv 2606.30015) and **Skill-to-LoRA** (arXiv 2606.16769): no public code.

## 7. Weight-space programming (MIT CSAIL, blog 2026-11-02)

- GPT-6 Astra edited 131,072 of ~0.8B weights of Qwen3.5-0.8B (subtracting a "loop direction" from 64 output columns of MLPs in layers 15 and 19): MATH-500 7.9→48.1%, GSM8K 13.8→67.1%, IFEval 21.0→62.3%; Gemma 3 270M no gain. https://rikiyat.github.io/blog/weight-space-programming/
- Logs + exact patch released: `manifest.json` (tensor names, shapes, sha256) + `edit.npz` (indices + bf16 values) + `apply_edit.py` → bit-exact rebuild → `.safetensors`. Runs took 8.5–20.3 h. https://github.com/RikiyaT/lm_understand-logs
- Applying the edit is reproducible; **producing** edits requires a frontier editor and the unreleased harness. Open-model replication so far covers only weight *reading* (Simon Willison, 2026-11-03).
- **This patch format (manifest + sparse diff + apply script) is a good template for a mod payload.**

## Runtime answer — what loads today without modifying the runtime

| Runtime | Loads as-is | Needs custom serving code |
|---|---|---|
| **llama.cpp / llama-server** | LoRA (multiple, hot-swap via `POST /lora-adapters`); control vectors (launch-time); merged-weight edits as new GGUF; prefix KV via slot save/restore (same prefix only) | Cartridges, KV Packet, non-prefix KV reuse, hot-swapped control vectors |
| **Ollama** | `ADAPTER` = one LoRA matching the exact base; merged-weight edited models as GGUF | Control vectors, cartridges, KV packets, stacked LoRAs |
| **vLLM** | One PEFT LoRA per request (dynamic load with `VLLM_ALLOW_RUNTIME_LORA_UPDATING`); merged-weight edits; non-prefix KV reuse via LMCache CacheBlend | Cartridges (Tokasaurus branch only), KV Packet, steering (fork/hooks), LoRA stacking |

**Implication for v0.** The only payload every runtime loads today is a LoRA. Cartridges — the best knowledge mechanism — need a loader we write (Tokasaurus fork, or a vLLM/llama.cpp patch). That loader *is* the product's technical core; see docs/risks.md #4.
