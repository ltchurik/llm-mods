# Corpora for the first mods — sources, licences, sizes, changelogs, evals

Researched 2026-11-07. Sizes marked *measured* come from shallow clones / word counts run that day; token estimates use ~1.3 tok/word (English prose), ~1.8 (changelogs/code-heavy), ~2.5 (Slovak). `sk-tokenizer-bench` gives exact Slovak numbers.

## Summary

| Corpus | Source | Format | Licence | Derived artefact OK? | Size est. | Changelog | Eval set | Latest |
|---|---|---|---|---|---|---|---|---|
| nginx docs + CHANGES | github.com/nginx/nginx.org · nginx.org/en/CHANGES | XML (custom DTD) · text | BSD-2-Clause | Yes | docs 154 files, 221k words (~300k tok) *measured*; CHANGES ~90k tok | Yes, per release | None | mainline 1.31.8 (2026-10-21), stable 1.30.6 |
| PostgreSQL 18 docs + release notes | postgresql.org/docs/18 · doc/src/sgml | DocBook | PostgreSQL License (MIT-like, covers docs) | Yes | A4 PDF 3,156 pages (~2M tok); release-18 notes ~15–18k words | Yes, incl. "Migration to Version 18" | motherduckdb/pg-docbench (text-to-SQL proxy) | 18.6 (2026-08-13) |
| Docker Engine 29 docs + RN; Compose spec | github.com/docker/docs · docs.docker.com/engine/release-notes/29 · compose-spec | Markdown (Hugo) | Apache-2.0 (repo); site footer "all rights reserved" → use the repo | Yes | engine 241k + reference 36k + compose 31k words; compose-spec ~60k tok | Yes, per patch with breaking-changes section | None | Engine 29.9.3 (2026-11-03); Compose v5.7.0; spec unversioned |
| Traefik 3.x docs + v2→v3 | github.com/traefik/traefik docs/content · doc.traefik.io/traefik/migrate/v2-to-v3 | Markdown (MkDocs) | MIT (repo) | Yes (site has no separate content licence) | 174 files, 192k words (~250k tok); migrate/ 17k words | Yes: migration guide + per-minor page + releases | None | v3.7.17 (2026-10-30); 3.8.0-rc1 |
| Kubernetes docs + CHANGELOG | github.com/kubernetes/website · kubernetes/kubernetes CHANGELOG | Markdown | Docs CC-BY-4.0; CHANGELOG Apache-2.0 | Yes (attribution; indicate changes) | content/en/docs 1.86M words (~2.4M tok); CHANGELOG-1.3x ~20–28k words each | Yes: "Urgent Upgrade Notes", "Changes by Kind" | SO K8s Q&A (not version-specific) | 1.37 (2026-08-26); 1.34 EOL 2026-10-27 |
| Terraform AWS provider 6.x | github.com/hashicorp/terraform-provider-aws website/docs | Markdown | MPL-2.0 | Yes (keep notice) | 2,757 files, 1.47M words (~1.9M tok); CHANGELOG 29k words | Yes, weekly releases + version-6 upgrade guide | IaC-Eval (458 scenarios, code-gen) | 6.72.0 (2026-11-04) |
| MCP specification | modelcontextprotocol.io/specification/latest · repo docs/specification | MDX + schema | MIT→Apache-2.0 (spec); CC-BY-4.0 (docs) | Yes | 2026-07-28 rev: 74.6k words (~100k tok); 2025-11-25: 52.5k; 2025-06-18: 30.3k | Yes, per-revision changelog + deprecations | None | 2026-07-28 (stateless redesign) |
| OpenAPI 3.2 | spec.openapis.org/oas/latest · OAI/OpenAPI-Specification | Markdown | Apache-2.0 | Yes | 3.2.0.md 37.8k words (~50k tok) | Appendix revision history; releases | None | 3.2.1 (2026-09-10) |
| EU AI Act (2024/1689) | eur-lex.europa.eu/eli/reg/2024/1689 · consolidated 02024R1689-20260727 | HTML/PDF/Formex | EUR-Lex: legal docs reusable (Decision 2011/833/EU); consolidated texts CC-BY-4.0 | Yes (attribute; OJ is only authentic) | consolidated EN 61k words (~80k tok); SK 53k words (~130k tok) | "Modified by" tables; amended by Reg. 2026/1744 (in force 2026-07-27) | AI Act Evaluation Benchmark arXiv 2603.09435 | consolidated 2026-07-27 |
| GDPR (2016/679) | eur-lex.europa.eu/eli/reg/2016/679 | HTML/PDF/XML | same | Yes | EN 55k words (~70k tok); SK 46k words (~115k tok) | corrigenda only | GenAIPABench (privacy QA) | consolidated 2016-05-04 |
| Slovak law: Zákonník práce 311/2001, zákon o DPH 222/2004 | static.slov-lex.sk/static/SK/ZZ/2001/311/20260901.html · …/2004/222/20260101.html (+ PDF, binding) | HTML + PDF | **Public domain by statute**: § 5 písm. b) zák. 185/2015 excludes legislative text from copyright | Yes | ZP 65k words (~160k tok SK); DPH 87k words (~215k tok SK) | Slov-Lex "História" time versions (no prose changelog; diffs must be generated) | None — candidate for sk-llm-eval | ZP eff. 2026-09-01 (next 2026-12-02); DPH eff. 2026-01-01 |
| Wikipedia Portal:Current events (monthly) | en.wikipedia.org/wiki/Portal:Current_events/<Month_Year> (daily subpages, `?action=raw`) | Wikitext | CC-BY-SA-4.0 | Yes, **ShareAlike** — derived cartridge likely must be BY-SA; flag for review | ~22k words/month (~30k tok) *measured* Oct 2026 | the corpus is a changelog | TiEBe (arXiv 2501.07482), evolveQA, DynaQuest | monthly |

## Notes per corpus

**nginx.** Two doc sets: nginx.org (open-source nginx, XML, BSD-2) and docs.nginx.com (F5 product docs, also BSD-2 since 2025). For a mod use nginx.org `xml/en/docs` (103 module reference pages) + CHANGES. Every directive carries `appeared-in="1.x.y"` in the XML — ideal for auto-generating "which directive appeared in version X" QA.

**PostgreSQL.** Docs are huge; scope a mod (SQL commands + config + release notes). Every 18.x minor has "Migration to Version 18.x". 18.6 notes are action-heavy (output_plugin_libraries whitelist, pgcrypto change). 18.7 expected ~2026-11-12.

**Docker.** 29.0.0 breaking changes: containerd image store default, default ulimit nofile 1024, API ≥ v1.44, cgroup v1 deprecated, Content Trust removed, `github.com/docker/docker` Go module deprecated. CLI reference is partly generated from docker/cli YAML — include `docker/cli/docs/reference` for exhaustive flags. Compose spec is unversioned; version-delta QA must come from docker/compose releases.

**Traefik.** MIT repo. v2→v3 guide + "Configuration changes for v3" + "v3 minor migrations" (9.5k words, 3.0→3.7). v3.8 at rc1.

**Kubernetes.** 1.37 is current; 1.34 went EOL 2026-10-27. CHANGELOG files (~20–28k words each) with "Urgent Upgrade Notes" and API removals are directly usable for "what changed between 1.34 and 1.35" QA. Version-pinned doc snapshots exist (v1-34.docs.kubernetes.io).

**Terraform AWS.** Per-resource "Argument Reference" tables are machine-parseable. Releases weekly → a mod ages fast; plan monthly rebuilds.

**MCP.** 2026-07-28 is a large break (stateless, no `Mcp-Session-Id`, no initialize handshake, `server/discover`, MRTR, `resultType`; Roots/Sampling/Logging deprecated). Prior revisions kept side by side → version-delta QA is easy to generate.

**OpenAPI.** 3.2.1 (2026-09-10). 3.0.x/3.1.x kept side by side.

**EU AI Act.** Use consolidated CELEX 02024R1689-20260727 (amended by Reg. 2026/1744, "Digital Omnibus on AI": Annex III high-risk date → 2027-12-02, Annex I → 2028-08-02), not the 2024 OJ text. Attribution required; only OJ is authentic.

**GDPR.** Unamended (corrigenda only); Digital Omnibus data proposals not yet reflected.

**Slovak law.** No licence to clear — legislative text is excluded from copyright by statute. e-Zbierka on Slov-Lex is legally binding; HTML carries an "informatívny charakter, záväzný je PDF" disclaimer → build from PDF or note it. Time-versioned URLs make "what changed between časová verzia A and B" diffs trivial. A third-party SK ELI MCP server exists (matematicsolutions/sk-eli-mcp). sk-llm-eval is the natural home for a Slovak-law QA set.

**Wikipedia current events.** ~30k tokens/month → cheap monthly mod. The one ShareAlike licence in the list — legal review before publishing a derived artefact. Daily subpages have consistent section headers and cite sources, which supports auto-QA.

## Version-specific QA datasets (for the eval harness)

- **VersiCode** (arXiv 2406.07411): version-specific code completion/migration, Python.
- **GitChameleon 2.0** (arXiv 2507.12367): 328 version-pinned Python problems with tests; frontier models ~48–51%.
- **LibEvoBench** (Google, ICML 2026, arXiv 2606.25402): PyTorch/NumPy/SciPy across versions; models are "version-oblivious"; adding docs +10–20 pts.
- **RustEvo²** (arXiv 2503.16922): 588 API changes; 56.1% pre-cutoff vs 32.5% post-cutoff; RAG +13.5 pts.
- **"When LLMs Lag Behind"** (arXiv 2604.09515): 270 real API updates; 42.55% executable without docs, 66.36% with.
- **EDAPIBench** (arXiv 2511.21022): deprecated APIs, 3,000+ edit instances.
- **evolveQA** (arXiv 2510.19172): QA from AWS/Azure update notes and WHO reports — closest analogue to changelog QA; up to 31% accuracy drop on evolving facts; in 32–45% of cases models give the outdated answer open-ended but pick the current one in MC.
- **RecencyQA** (arXiv 2603.16544): 4,031 questions labelled by answer-change frequency.
- **IaC-Eval** (NeurIPS 2024): 458 AWS Terraform scenarios; best model 19.36% pass@1.
- **TiEBe** (arXiv 2501.07482): Wikipedia-events recall over time.

**Gap**: nothing exists for "which nginx/k8s/postgres flag appeared in version X". That is both cheap to generate (nginx `appeared-in`, PG release notes, K8s CHANGELOG) and the launch benchmark for the store.

## Evidence that LLMs confuse versions

- ICSE 2025 "LLMs Meet Library Evolution" (arXiv 2406.09834): 7 LLMs, 145 API mappings, 28,125 prompts; deprecated-API usage 25–38%.
- LibEvoBench: 5–10 pt drop on evolving vs stable APIs; 6–14% anachronistic-parameter errors; explicit version tag gives no gain; docs in context +10–20 pts — the direct argument for a knowledge mod from docs.
- evolveQA and RustEvo² numbers above.
- IaC: LLM-generated K8s YAML 35.8% with quality issues; `apps/v1beta1` and TF AWS provider v4 vs v5 attribute drift (tianpan.co, 2026-05-06).
