# llm-mods

Research and design for **mods for open language models**: installable packages of knowledge or behaviour that a user drops onto a local model — no training on their side — composable, versioned per base model, verified before publish. Think Minecraft mods / Modrinth, for LLMs.

Status: **research stage** (started 2026-11-07). No code yet; this repo holds the research, the product spec and the MVP plan. Code lands here when the plan survives its own kill test.

## Why

Open models have frozen knowledge, blend tool versions, and are thin on small languages and niche domains. The only no-training fix today is RAG, which is infrastructure, not a package. In 2026 the research for putting knowledge *into* a frozen model without user-side training became real and open — trained KV-cache cartridges, hypernetwork-generated LoRAs, closed-form knowledge edits — but nobody packages it with an install command, compatibility rules, composition rules or a quality gate. That gap is this project.

## Contents

| file | what |
|---|---|
| [docs/background.md](docs/background.md) | the enabling research, why now, adjacent ideas we rejected |
| [docs/mechanisms.md](docs/mechanisms.md) | deep-dive per mechanism: cost, size, quality, composition, runtime support, licence |
| [docs/ecosystem.md](docs/ecosystem.md) | runtime support (vLLM, SGLang, llama.cpp, Ollama, PEFT), existing registries, RAG vs parametric, audience size, base-model churn |
| [docs/corpora.md](docs/corpora.md) | candidate corpora for the first mods: sources, licences, sizes, changelogs, evals |
| [docs/first-mods.md](docs/first-mods.md) | which knowledge to package first and why |
| [docs/risks.md](docs/risks.md) | ranked risks, kill criteria, open questions |
| [spec/product-spec.md](spec/product-spec.md) | users, mod format, manifest, loader, maker, store |
| [spec/mvp-plan.md](spec/mvp-plan.md) | 4-week plan, scope decisions, kill test |

## The short version

- **Mod** = payload (cartridge / LoRA / edits / steering vector) + manifest (base-model hash, mechanism, claims, budget, before/after eval, licence, signature).
- **`modctl install`** stacks mods onto a local model; refuses mismatched bases; applies composition rules.
- **`modctl make --from docs/`** turns a corpus into a mod on a maker's GPU in minutes.
- **Store** = static per-base catalog with verified badges; private stores later.
- **v0** = cartridges only, one base model, 10 mods, one benchmark post, one kill test against RAG.

## Licence

Apache-2.0.
