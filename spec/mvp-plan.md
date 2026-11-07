# MVP plan — 4 weeks, time-boxed bet

Goal: prove that installable knowledge mods beat RAG on something concrete for local-model users, with 10 real mods and a working loader. If they don't, keep the mods as plain adapters and park the loader.

## Scope decisions

| decision | choice | why |
|---|---|---|
| mechanism for v0 | cartridges (trained KV cache) only | no per-base hypernetwork needed; composable at inference; code is open |
| base model | one small open model with decent multilingual coverage (final pick after runtime research; candidates Qwen3.5-4B / Gemma 4 small) | one base keeps the compatibility matrix trivial |
| runtime | vLLM first (prefix/KV injection path), llama.cpp second if its state-load API can carry a cartridge | see docs/ecosystem.md |
| maker hardware | DGX Spark (GB10, 128 GB unified) | self-study on a 300k-token corpus is minutes, not hours |
| store | static: this repo + `index.json`, one dir per mod | no backend until there are makers other than us |

## Weeks

### Week 1 — loader skeleton + first cartridge
- `modctl` CLI (Go or Python; decide by runtime bindings) with `install / list / remove / verify / serve`
- manifest schema + validator (base hash, tokenizer hash, budget)
- make the first cartridge by hand with the upstream self-study code: `nginx-1.29`
- eval harness: 100 Q/A pairs per mod, exact/contains match; plus a fixed retention set (IFEval subset + 100 general QA)
- publish before/after numbers

### Week 2 — maker + 5 mods
- `modctl make --from <dir>` wrapping self-study: chunking, synthetic-QA generation, training, eval, manifest
- mods: `postgres-18`, `docker-29`, `traefik-3`, `mcp-spec`, `openapi-3.2`
- composition tests: pairs and triples; record conflicts and budget limits in manifests

### Week 3 — second runtime + repo-as-mod demo
- llama.cpp/Ollama path if feasible; otherwise document why not
- `modctl make --from ./repo` demo on a public codebase (the onboarding story)
- mods: `eu-ai-act`, `gdpr`, one changelog mod (`postgres-17-to-18`)

### Week 4 — the kill test + launch
- side-by-side on 5 task sets: plain model vs RAG (same corpus, standard retriever) vs mod vs mod+RAG
- metrics: accuracy, latency, tokens/query, works-offline, setup steps
- write-up with charts; Show HN + blog post
- decision: continue to v1 (Text-to-LoRA skills, edits, signing, web store) or park

## Kill criteria (decided before building)

Park the loader if, on the week-4 test, mods do not win on at least one of: (a) accuracy on cross-document questions, (b) latency per query at equal accuracy, (c) accuracy with no network. "Roughly equal to RAG" is a loss — RAG is simpler.

## Deliverables

- `ltchurik/llm-mods` (this repo): research, spec, loader, maker, store index, mods
- 10 mods with manifests and eval badges
- one benchmark post: "Does your local model know Postgres 18? Before/after"
