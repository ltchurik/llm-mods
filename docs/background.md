# Background — why now

Written 2026-11-07. The three pieces of 2026 research that make "knowledge without user-side training" practical, plus the adjacent landscape we checked and rejected.

## The enabling research

| work | what it showed | relevance |
|---|---|---|
| **Cartridges** (Hazy Research, ICLR 2026) | A corpus can be distilled offline into a small trained KV cache; load it and the model answers as if the corpus were in context, at ~38× less memory. Independently trained cartridges compose at inference. | The v0 mod payload. |
| **Cartridges++** (arXiv 2609.35621) | Names the failure: as a cartridge grows it degrades off-context knowledge and instruction following; proposes a fix. | Defines the "budget" field and the retention eval. |
| **Doc-to-LoRA / Text-to-LoRA** (Sakana AI, ICML 2026) | A meta-trained hypernetwork emits a LoRA from a document or a task description in one forward pass; no gradient steps at deployment. Hypernetwork is per base model. | v1 mechanism for *skills* (behaviour mods) and instant doc mods where a hypernet exists. |
| **AlphaEdit / UltraEdit / EasyEdit** | Insert or override thousands of facts by closed-form weight edits, preserving most general ability; reported to fail on some model families and past ~5k edits. | Small "fact patch" mods (terminology, corrections). |
| **Sparks of Weight-Space Programming** (MIT CSAIL, 2026-11-02) | A frontier model read a small model's raw weights and patched 0.016% of them to fix reasoning loops. | Future "behaviour patch" mod type; not reproducible without frontier access today. |
| **Language Models Need Sleep** (Google, 2026-10) | Wake/sleep continual learning: consolidate context into weights offline, then "dream" on self-generated data. | The maker's offline step is a sleep phase; the nightly-distill daemon idea is the same loop as a product. |

## Why "mods" and not "fine-tunes" or "RAG"

- Fine-tunes need the user to train. Mods are made once by a maker; the user installs.
- RAG needs infrastructure at query time (embedding, vector store, retriever) and fails on cross-document questions and offline use. A mod is a file.
- Adapters on Hugging Face are files too, but with no compatibility rules, composition rules, budget, or quality gate — the same situation Minecraft modding was in before mod loaders and Modrinth.

## Adjacent ideas we looked at and why they are not the product

| idea | status found | verdict |
|---|---|---|
| Public LLM drift monitor | several exist (model-drift, DriftWatch, daily-bench, SEISMOGRAPH) | taken |
| MCP server trust registry / dynamic scanner | crowded (MCP Skills, Dominion Observatory, AddonAudit, MCPZoo, Cisco, mpak) | taken; mcp-jail stays the enforcement layer |
| Notarised archive of AI outputs | notaries exist (x402 Notary, VAOL, CertifiedData) but trust the client's claim | thin gap, different audience |
| Language token-tax leaderboard | exists (41-language dataset on Zenodo, African-language leaderboard) | taken |
| AI privacy-policy change tracker | exists (TOSTracker, AI privacy tracker) | taken |
| "Git for weights" / community merging | Ballast (versioning), SLAO, federated merging literature | versioning exists; *evidence-gated merge* ("Weight CI") is open and is a natural v2 of the store's verification pipeline |
| Nightly distill daemon | no shipped product found | sibling product; same maker engine |

## Where this sits relative to the Spark

The DGX Spark (GB10, 128 GB unified) is the maker box: self-study for a cartridge on a few-hundred-thousand-token corpus is minutes; continued-pretraining deltas for a language pack (1–10 B tokens on a 3–4 B model) are hours to days. Users never need it.
