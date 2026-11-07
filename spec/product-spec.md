# Product spec — mods for open models

Status: draft v0.1 (2026-11-07). Research-stage; nothing here is implemented yet.

## One sentence

Install knowledge and behaviour into a local open model like a game mod: no training on the user's side, composable, versioned per base model, verified before publish.

## Problem

- Open models ship with frozen knowledge, mixed tool versions, thin coverage of small languages and niche domains.
- The only "no training" option today is RAG: a vector DB, a retriever, chunking, and retrieval misses. It is infrastructure, not a package.
- The mechanisms that put knowledge *into* a model without user-side training now exist as research code (trained KV-cache cartridges, hypernetwork-generated LoRAs, knowledge edits, steering vectors) but nobody packages them: no install command, no compatibility rules, no composition rules, no quality gate, no catalog.

## Users

1. **Installer** — runs a local model (Ollama / llama.cpp / vLLM) and wants it to know *X* offline: a tool version, a spec, a law, their codebase. Never trains anything.
2. **Maker** — has a corpus and a GPU box; produces a mod in minutes and publishes it.
3. **Operator** (later) — runs a private store for a team (runbooks, internal docs).

## Core objects

### Mod

A directory or archive with:

```
manifest.json      # identity, base-model binding, mechanism, claims, evals, licence, signature
payload/           # cartridge.safetensors | adapter.safetensors | edits.jsonl | steer.safetensors
eval/              # questions + expected answers used for the badge (optional but required for "verified")
README.md
```

### Manifest (draft fields)

| field | meaning |
|---|---|
| `name`, `version` | semver; `nginx-1.29` + `1.0.3` |
| `mechanism` | `cartridge` \| `lora` \| `kv-packet` \| `edit` \| `steer` |
| `base.model`, `base.sha256` | exact checkpoint the payload was made for |
| `base.tokenizer_sha256` | tokenizer must match |
| `runtime` | minimum runtime versions that can load it |
| `claims[]` | human-readable: "answers nginx 1.29 directive questions" |
| `budget` | memory / context cost when loaded (KV positions, adapter params) |
| `compose.conflicts_with[]`, `compose.max_stack` | composition rules learned from eval |
| `eval.before`, `eval.after`, `eval.retention` | scores on claim set and on a fixed general set |
| `source.corpus`, `source.licence` | provenance of the knowledge |
| `author`, `signature` | ed25519 over payload+manifest |

### Loader (`modctl`)

```
modctl install nginx-1.29 postgres-18 ./my-repo-mod
modctl list / remove / verify
modctl serve   # starts the runtime with the mod stack applied
```

Behaviour: refuse to install a mod whose base hash does not match the loaded model; warn on budget overflow; apply composition rules.

### Maker (`modctl make`)

```
modctl make --from docs/ --base qwen3.5-4b --mechanism auto
```

Picks the mechanism by content type (facts/corpus → cartridge; style/skill → text-to-lora; small fact set → edits), runs self-study / hypernetwork / edit on the local GPU, runs eval, writes manifest.

### Store

Static registry first (GitHub repo + index.json), per base model. Each listing: compatibility matrix, eval badges, size/budget, licence, changelog. Verification pipeline runs the mod's eval and the retention eval before a listing gets the "verified" badge.

## Non-goals (v0)

- Teaching a model a language it does not know (needs pretraining-scale data).
- Replacing RAG for private, fast-changing data (per-minute changes).
- Supporting closed/API models.

## Success criteria for the MVP

- `modctl install` works on one base model in two runtimes.
- 10 published mods with before/after eval badges.
- Side-by-side vs RAG on 5 tasks: mods win on at least one of offline quality, latency, or composition; otherwise the loader is parked and only the mods (as plain adapters) are kept.
