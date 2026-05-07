# LLM Wiki Plugin — Decisions Log & Version History

Companion to [DESIGN.md](DESIGN.md). The decisions log records *why* the design landed where it did; the version history records when each piece shipped.

## Decisions Log

A condensed record of the design decisions made while shaping the plugin, with what we considered and what we picked.

| # | Decision | Considered | Chosen | Why |
|---|---|---|---|---|
| 1 | Implementation form | Skill / desktop app (à la nashsu) / web app + MCP (à la lucasastorian) | **Skill** | Closest to Karpathy's gist; 100× less code; user already disliked the desktop app; manual workflow was already producing value |
| 2 | Per-vault config | Single `CLAUDE.md` (gist-faithful) / `purpose.md` only / **`purpose.md` + `schema.md`** | **`purpose.md` + `schema.md`** | Clean why/how separation; matches downstream convention (nashsu); semantic clarity |
| 3 | v1 scope | A (workflow only) / **B+C+D** (workflow + bootstrap + ingest helpers + qmd) | **B+C+D** | User explicitly opted into all of it; matches the full Karpathy gist |
| 4 | Plugin packaging | Skill only / **plugin with 1 skill + slash commands** / slash-commands only | **Plugin: 1 skill + 4 slash commands + 1 helper** | Skill provides reusable workflow; slash commands give explicit ergonomics; both invocation paths converge |
| 5 | Bootstrap syntax | `/llm-wiki:init <name>` (minimal) / `<name> <scenario>` / **`<name> <scenario> "<context>"`** (free-form) | **3-arg form** | Optional free-form context unlocks one-shot rich init via LLM call; minimal 1-arg form still works |
| 6 | Scenarios shipped | 1 (research only) / 2 / **5 (research, reading, personal, business, general)** | **5** | Covers Karpathy's enumerated use cases; matches nashsu's choice |
| 7 | Vault detection | Marker dir `.llm-wiki/` / **`purpose.md` presence** / explicit cli flag every time | **`purpose.md` presence** (with `--vault` override) | One less file; semantic ("a vault is something with a purpose"); cwd-walking to git root works |
| 8 | Batch ingest | Out of scope / always-confirm batch / autonomous batch | **Always-confirm batch** (Karpathy-faithful) | "Supervised one at a time" is a feature, not a bug; smart batch (find unprocessed) reduces friction without skipping the pause |
| 9 | Idempotency | None / **SHA-256 cache** | **SHA-256 cache** | Saves LLM calls on re-ingest; `files_written` enables cascade-delete |
| 10 | qmd | Bundled / **optional, threshold-based** / never integrate | **Optional, threshold-based** | Karpathy explicitly recommends qmd; gracefully fall back below 100 pages |
| 11 | qmd integration mode | CLI only (v1) / MCP-native (deeper integration) | **CLI only in v1**, MCP optional v1.5 | Works everywhere immediately; MCP gives marginal latency gain not worth the v1 complexity |
| 12 | Cross-vault | Per-vault only / **registry + cross-vault query** | **Registry + cross-vault query** (lint stays per-vault) | User explicitly wanted; cross-vault lint defers due to schema.md scoping |
| 13 | Cross-vault citation | `[[wiki-name/page]]` / **`[[wiki-name:page]]`** | **`[[wiki-name:page]]`** | User preference; colon clearly separates wiki-name from slug |
| 14 | Distribution | Local-only / **marketplace-ready from start** | **Marketplace-ready** | Adds 2 small JSON files; no extra effort; future-proofs sharing |
| 15 | MCP server | Build one / **none for v1** | **None for v1** | Filesystem + Bash tools cover everything; MCP would just be a layer; qmd's MCP can be wired later |
| 16 | Add `/llm-wiki:research`? | Defer to post-smoke-test / fold into v0.1.0 now | **Fold into v0.1.0 now (released as v0.2.0)** | User asked; design was clear from prior discussion; infra reuse is high (firecrawl, ingest pipeline, cache, registry already present) |
| 17 | Source-selection model for research | LLM picks autonomously / **Always-ask-first** | **Always-ask-first** | Karpathy's pattern is "human curates"; LLM-rated star list is a triage assist, not a delegation |
| 18 | Per-source pause in research mode | Always supervised / Always batch / **Configurable** | **Configurable: supervised default, `--unsupervised` to opt out** | Default to Karpathy-faithful; let power users speed up |
| 19 | Command filename prefix | `wiki-init.md` etc. / **`init.md`** etc. | **Drop `wiki-` prefix** (v0.2.3) | Commands are namespaced by the plugin (`/llm-wiki:init`) so the filename prefix was redundant; shorter names cleaner |
| 20 | purpose.md auto-draft on unfilled template | Warn only / **Offer to draft** | **Offer to draft in Step 0.5** (v0.3.0) | Weak triage from an empty `purpose.md` was a known pain; surfacing the draft offer at research time fixes the issue in-flow |
| 21 | Add `/llm-wiki:rm`? | Defer / **Add as slash command** | **Add as slash command** (v0.4.0) | User needed a safe way to delete vaults; hard safety checks + atomic registry update make it reliable |
| 22 | Drift detection in ingest | Silent / Auto-update / **Always-ask-first** | **Step 5.5 with `y/n/s` prompt** (v0.4.0) | `purpose.md` is owned by the user; the skill may suggest but never auto-save. `s` option lets users permanently skip noisy sources |
| 23 | 9th lint check (purpose drift) | Auto-fix / **Flag as suggestion** | **Suggestion severity, not auto-fixable** (v0.4.0) | Resolution requires deciding whether to update `purpose.md` or relocate sources — not safe to automate |
| 24 | `/llm-wiki:init` path UX | Status quo (cwd-default, document better) / **Confirm bare names** / `~/wikis/<name>` default | **Prompt on bare names; explicit `/` or `~` paths skip the prompt** (v0.4.1) | Real failure mode: user runs from `~`, vault lands at `/Users/f.luo/<name>` unintentionally. Prompt is one-shot, no behavior change for users who pass explicit paths |
| 25 | qmd lifecycle ownership | User runs `qmd collection add/update/embed` themselves (gist-faithful) / Skill registers eagerly on init / **Skill registers lazily on first ingest, updates after each ingest/lint, removes on rm** | **Lazy registration + auto update/embed/remove** (v0.4.2) | "Optional and managed-for-you" matches the user expectation set by `/llm-wiki:init`'s install hint; lazy beats eager because qmd may be installed after a vault already exists; old `qmd index --update` syntax also didn't exist in qmd 2.1, so the rewrite was forced anyway |
| 26 | PDF extraction primary | Read tool (status quo) / **Three-tier chain: marker → Read tool → pdftotext** / Mistral OCR (paid cloud) | **Three-tier chain with marker preferred when installed** (v0.4.3) | First real ingest test exposed Read-tool quality issues on arxiv PDFs (multi-column, equations); marker is OSS, local, preserves LaTeX equations and Markdown tables. Made it preferred-when-available rather than required, so vaults without marker keep working unchanged. Mistral OCR rejected for v1 because it's paid+cloud and marker is good enough for research papers |

## Version History

| Tag | Date | Highlights |
|---|---|---|
| v0.1.0 | 2026-05-06 | Initial release: skill + 5 slash commands per Karpathy's gist |
| v0.2.0 | 2026-05-06 | Adds `/llm-wiki:research` (web search + LLM triage + batch ingest) |
| v0.2.1 | 2026-05-06 | Bug fixes from smoke test (cache format prose, lint check 8 scope, system-file dangling exemption) |
| v0.2.2 | 2026-05-06 | Plugin install fix (move marketplace.json to canonical `.claude-plugin/` location with correct schema) |
| v0.2.3 | 2026-05-06 | Drop redundant `wiki-` prefix from command filenames; commands are now `/llm-wiki:init` etc. |
| v0.3.0 | 2026-05-06 | `/llm-wiki:research` offers to draft `purpose.md` from the topic when the vault's `purpose.md` is still a template |
| v0.4.0 | 2026-05-06 | `/llm-wiki:rm` for safe vault deletion; ingest offers `purpose.md` updates on scope drift; lint adds purpose-drift check (9th check) |
| v0.4.1 | 2026-05-06 | `/llm-wiki:init` confirms the resolved path before scaffolding when a bare name is given; redirect to a different `/`- or `~`-prefixed path inline without re-running |
| v0.4.2 | 2026-05-06 | Skill auto-manages qmd collection lifecycle (lazy register on first ingest, update + embed after each ingest/lint/research, remove on `/llm-wiki:rm`); query path uses `qmd query -c <vault>` (hybrid + rerank) instead of the previous `qmd search`. Users no longer touch qmd directly |
| v0.4.3 | 2026-05-06 | PDF extraction prefers `marker_single` (marker-pdf) when available — preserves equations as LaTeX, tables as Markdown, multi-column layouts. Falls back to Read tool, then `pdftotext`. Vaults without marker installed see no behavior change |
