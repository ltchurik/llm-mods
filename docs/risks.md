# Risks and open questions

Ordered by how likely each is to kill the project.

| # | risk | why it matters | mitigation / test |
|---|---|---|---|
| 1 | **RAG is good enough** | "Knowledge without training" already exists for everyone via retrieval. If mods only match RAG, nobody switches. | Week-4 kill test (see spec/mvp-plan.md). Target the cases RAG is bad at: cross-document reasoning, offline, many small facts, version pinning. |
| 2 | **Base-model churn** | Every mod binds to one checkpoint; small open models ship monthly. Store fragments like game versions. | Per-base catalog from day one; automated rebuild pipeline for the maker's own mods; measure rebuild cost per base. |
| 3 | **Quality ceiling of cartridges** | Cartridges++ reports that larger cartridges reduce off-context knowledge and instruction following. A mod that makes the model dumber kills trust. | Budget field in manifest; retention eval mandatory for "verified"; adopt Cartridges++ fixes. |
| 4 | **Runtime integration** | If vLLM/llama.cpp cannot load a trained KV prefix without patches, "install" means "patch your runtime". | Research in docs/ecosystem.md; pick the runtime with the cleanest path; contribute upstream if small. |
| 5 | **Hypernetwork coverage (Doc-to-LoRA)** | Hypernets exist for few bases; meta-training one is real ML work. | v0 = cartridges only; Text-to-LoRA skills as v1 only where hypernets exist. |
| 6 | **Platform risk** | vLLM/Ollama/HF could ship "load cartridge" natively; HF Hub already hosts adapters. | Moat = format + verification + catalog + makers. Move fast; make the format runtime-neutral so natives become backends. |
| 7 | **Licensing of corpora** | Docs/legal texts have varied licences; a derived artefact may or may not be redistributable. | docs/corpora.md tracks licence per corpus; prefer permissive/official-reuse sources; manifests carry `source.licence`. |
| 8 | **Small market** | Local-LLM users wanting installable knowledge: tens of thousands. | Fine for OSS traction and brand; business angle (private team stores) is a later question. |
| 9 | **Eval gaming / trust** | A maker can tune to its own eval set. | Verified badge requires hidden eval split run by the store; signatures; reproducible make scripts. |
| 10 | **Composition limits unknown** | How many cartridges/adapters stack before interference? | Measure in week 2; record `compose.max_stack` per base. |

## Open questions

- Which base model: multilingual coverage vs ecosystem (hypernets, GGUF availability) vs size.
- Cartridge sizes at which composition still works (positions per layer per mod).
- Can a cartridge be converted to a GGUF-loadable state for llama.cpp, or does it need a runtime patch?
- Signing and provenance: reuse sigstore? mirror what mcp-jail does for policy and audit?
- Private stores: static index + object storage is probably enough for v1.
