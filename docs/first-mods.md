# First mods — what knowledge to package

Filter for a good mod (all five, ideally):

1. Base models get it wrong confidently (mixed versions, post-cutoff, non-English, niche).
2. It changes on a schedule, so versioned mods make sense.
3. Answers need many small facts across a corpus — where retrieval of 5 chunks fails.
4. Corpus is open-licensed and has (or can get) an automatic eval.
5. Someone running a local model wants it.

## Ranked families

| family | first mods | why mods > RAG | eval | notes |
|---|---|---|---|---|
| Versioned tool docs | `nginx-1.29`, `postgres-18`, `docker-29`, `traefik-3`, `k8s-1.34`, `terraform-aws-6` | models blend versions; a mod pins one; offline on servers | 100–200 directive/flag/config Q/A per mod generated from the docs, human-checked | our domain; the launch benchmark |
| Changelog / breaking-change mods | `postgres-17-to-18`, `docker-28-to-29`, `traefik-2-to-3` | "what breaks if I upgrade" is cross-document reasoning | known breaking-change lists | revives the breaking-change diff idea |
| Specs & standards | `mcp-spec`, `openapi-3.2`, `rfc-9110..9114` | dense, cross-referenced; whole-spec answers | conformance questions | overlaps the mcp-jail audience |
| Law & regulation | `eu-ai-act`, `gdpr`, `sk-zakonnik-prace`, `sk-dph` | many interacting articles; offline for law firms; non-English | Q/A from official explanatory memos | Slovak sources + sk-llm-eval |
| Local / national knowledge | `sk-institutions-2026`, `cz-*` | base models are thin on small countries | Wikipedia / official data | corpus from our portals |
| Your own codebase / runbooks | `modctl make --from ./repo` | private, offline, no vector DB | repo Q/A | onboarding demo |
| Monthly world update | `world-2026-11` | post-cutoff facts | dated Q/A | Wikipedia current events only (licence-clean) |
| Language packs | `sk-core`, `cs-core` | fluency uplift; composable with the above | sk-llm-eval | slowest; needs a trained delta made once by the maker |

## Skip

Medical, personal finance (liability, weak eval), paywalled docs, "all of Wikipedia" (that is pretraining, not a mod).

## Build order

1. Versioned tool docs + one changelog mod (week 1–2).
2. Repo-as-mod demo (week 3).
3. EU AI Act + GDPR (week 3).
4. Slovak law + `sk-core` (after the kill test passes).

Licences, sizes and eval sources per corpus: see `docs/corpora.md`.
