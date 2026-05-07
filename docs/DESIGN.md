# LLM Wiki Plugin — Design

> Status: implemented (v0.4.0). For the decisions log and version history, see [DECISIONS.md](DECISIONS.md).

## Executive Summary

A Claude Code plugin that turns Karpathy's [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) into a reusable skill. Lets you bootstrap, maintain, query, research, and lint *plain-markdown* knowledge bases — one per research topic, life area, or project — directly from Claude Code, with Obsidian as the human-side IDE.

**v0.4.0 scope** (the canonical state documented below) is the full Karpathy spec implemented as a skill: pattern-level workflows in the skill, per-vault config in `purpose.md` + `schema.md`, seven slash commands (`/llm-wiki:init`, `/llm-wiki:ingest`, `/llm-wiki:query`, `/llm-wiki:research`, `/llm-wiki:lint`, `/llm-wiki:list`, `/llm-wiki:rm`), built-in document extraction (PDF / DOCX / PPTX / XLSX), SHA-256 idempotency cache, smart batch ingest, optional [qmd](https://github.com/tobi/qmd) hybrid search at scale, cross-vault query via a registry of known wikis, web-search-based deep research with LLM triage, purpose.md auto-draft offer, and drift detection during ingest and lint.

**Distribution:** Claude Code plugin, marketplace-ready (`marketplace.json` + `plugin.json`), but installs locally first.

**What it explicitly is not:** a desktop app, a hosted service, an MCP server (none required for v1), or a daemon. The skill runs while you're in a Claude Code session — same workflow as today, just encoded.

---

## Table of Contents

1. [Background — The LLM Wiki Pattern](#part-1--background--the-llm-wiki-pattern)
2. [Skill Design](#part-2--skill-design)
3. [Non-Goals / Deferred to Future Versions](#part-3--non-goals--deferred-to-future-versions)
4. [Acceptance Criteria](#part-4--acceptance-criteria)
5. [Open Risks](#part-5--open-risks)

---

# Part 1 — Background — The LLM Wiki Pattern

## 1.1 The Problem

The default LLM-with-documents workflow is **RAG**: upload sources, ask a question, the system retrieves chunks at query time, the LLM synthesizes an answer. NotebookLM, ChatGPT file uploads, and most chatbot-over-docs products work this way.

Three things go wrong with RAG-as-default:

1. **Knowledge doesn't compound.** Each query re-derives synthesis from raw chunks. The 50th query is no faster, no smarter, and no more aware of cross-document connections than the 5th.
2. **Cross-document analysis is rebuilt every time.** Asking "how does X compare to Y across these five papers?" forces the LLM to find and piece together fragments fresh, even if you've asked similar questions before.
3. **Maintenance is the human bottleneck.** Hand-maintained wikis (Notion pages, personal Obsidian vaults) need someone to update cross-references, fix stale summaries, merge overlapping notes. That work scales with sources, and people stop doing it.

The pattern Karpathy describes inverts this. **Synthesis happens at ingest time and gets written to disk as Markdown.** Each query reads the already-compiled wiki, not the raw sources. Cross-references are pre-computed; contradictions are pre-flagged. The wiki *compounds*.

The maintenance problem is solved by giving the maintenance work to the LLM, which "doesn't get bored, doesn't forget to update a cross-reference, and can touch 15 files in one pass."

## 1.2 Karpathy's Pattern

The original gist is a 76-line abstract pattern. The structure:

### Three layers

```
┌─────────────────────────────────────────────────────┐
│  Raw sources (immutable)                            │
│  PDFs, articles, transcripts, web clips, notes      │
│  ─────────────────────────────────────              │
│  The LLM reads from this layer; never modifies it.  │
└─────────────────────────────────────────────────────┘
                         │
                         │ ingest (LLM reads → writes)
                         ▼
┌─────────────────────────────────────────────────────┐
│  Wiki (LLM-maintained markdown)                     │
│  Per-source summaries, entity pages, concept pages, │
│  comparisons, an evolving synthesis. Cross-linked   │
│  via Obsidian-style [[wikilinks]].                  │
│  ─────────────────────────────────────              │
│  The LLM owns this layer. The human reads.          │
└─────────────────────────────────────────────────────┘
                         │
                         │ governed by
                         ▼
┌─────────────────────────────────────────────────────┐
│  Schema (CLAUDE.md / AGENTS.md)                     │
│  Tells the LLM how the wiki is structured,          │
│  what conventions apply, what workflows to follow   │
│  for ingest/query/lint.                             │
└─────────────────────────────────────────────────────┘
```

### Three operations

- **Ingest** — drop a source into raw, LLM reads, summarizes, updates entity/concept pages, appends a log entry, touches 5–15 files.
- **Query** — ask a question; LLM reads `index.md` first, drills into relevant pages, synthesizes with `[[wikilink]]` citations. Non-trivial answers can be filed back as new wiki pages.
- **Lint** — health check: contradictions, stale claims, orphan pages, missing concept pages, outdated synthesis.

### Two anchor files

- **`index.md`** — content catalog. Every page listed with a one-line summary. The LLM reads this *first* when answering a query, then drills in.
- **`log.md`** — append-only chronological record. Format `## [YYYY-MM-DD] type | title` makes it grep-friendly. Gives both the human and the LLM a timeline.

### Operating philosophy

> *"Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase."*

The human is open in Obsidian (graph view, plugins, manual editing). The LLM is open in a chat / agent UI. They edit the same Markdown files. The maintenance burden is gone because the LLM does it.

## 1.3 Surveyed Implementations

We examined three concrete realizations of the pattern, plus our own manual workflow:

### 1.3.1 Karpathy's gist (the spec)
- 76 lines, deliberately abstract
- Doesn't prescribe directory layout, page-type names, scenario presets, or specific tools beyond Obsidian + git
- Recommends: Obsidian Web Clipper, attachment folder, graph view, Marp / Dataview plugins, [qmd](https://github.com/tobi/qmd) for search at scale
- Single config file: `CLAUDE.md` (or `AGENTS.md` for Codex)

### 1.3.2 Our manual workflow (`~/git/llm-world-models-wiki/`)
- Built collaboratively in this conversation
- Currently: 17 stub pages spanning concepts/entities/sources/questions/comparisons, plus index.md / log.md / synthesis.md
- One CLAUDE.md (110 lines) holds workflow + domain
- Closest implementation to the gist's intent: terminal Claude Code + Obsidian side-by-side, plain markdown, no infra
- This was the **inspiration for the skill**: the workflow already works manually; we want to encode it so it ports to other research topics without re-bootstrapping

### 1.3.3 LLM Wiki desktop app — `nashsu/llm_wiki`
- Tauri (Rust + React 19) desktop app, ~68k LOC
- Implements the gist as a productized standalone app
- **Notable additions to the gist:**
  - `purpose.md` as separate "wiki's soul" file (domain + headline question)
  - 5 scenario presets (Research, Reading, Personal, Business, General)
  - Two-stage Chain-of-Thought ingest (analyze → generate)
  - 4-signal knowledge graph (link weight, source overlap, Adamic-Adar, type affinity)
  - Louvain community detection on the graph
  - Built-in PDF/DOCX/PPTX/XLSX extraction (pdfium, pandoc, calamine)
  - LanceDB vector search
  - Persistent ingest queue with crash recovery
  - Asynchronous review queue (LLM flags items for human judgment)
  - Chrome extension web clipper
- **Trades:** replaces Obsidian as the IDE with its own Next-React UI; heavier infrastructure (~5.6k LOC Rust backend + 50k LOC TS frontend); cumbersome to configure (we tried — the LLM provider setup, ingest, and review steps had real friction)
- Detailed analysis: see [`~/src/doc/llm_wiki-architecture.md`](../../../llm_wiki-architecture.md) and [`~/src/doc/llm_wiki-reading-plan.md`](../../../llm_wiki-reading-plan.md)

### 1.3.4 lucasastorian/llmwiki
- Python (FastAPI) + Next.js + SQLite + MCP server, ~12k LOC, ~821 GitHub stars
- **Notable choices:**
  - Filesystem is source of truth; SQLite (FTS5) is a derived index
  - MCP server exposes search/read/write tools to Claude Desktop or Claude Code
  - One workspace = one MCP server entry (intentional scoping)
  - pdf-oxide for PDF extraction, optional Mistral OCR
  - Custom Next.js UI (replaces Obsidian)
  - Hosted multi-tenant version at llmwiki.app (Postgres + Supabase + S3)
- **Trades:** also replaces Obsidian with custom UI; needs daemon (FastAPI + Next.js) running; multi-tenant version adds infrastructure complexity

### 1.3.5 Where each sits relative to the gist

| Implementation | Relationship to gist | Effort to build | Effort to maintain |
|---|---|---|---|
| Manual (ours) | ✓ literally Karpathy's described setup ("LLM agent on one side, Obsidian on the other") | hours | hours/year |
| `nashsu/llm_wiki` | replaces Obsidian with custom desktop UI; adds vector search, queue, web clipper, OCR | months | months/year |
| `lucasastorian/llmwiki` | replaces Obsidian with custom web UI; adds MCP server, FTS5 index, hosted mode | weeks/months | weeks/year |
| **This skill (proposed)** | ✓ literally Karpathy's setup, made portable as a Claude Code plugin | days | days/year |

The first column rows are roughly stack-ranked from "least infrastructure / most aligned with the gist" to "most infrastructure / most diverged". The skill design sits at the top of that ranking by intent.

## 1.4 Our Refinements

We're not implementing the gist verbatim. We adopt the following refinements (most picked up from `nashsu/llm_wiki` or invented during design), each justified inline:

| Refinement | Rationale | Origin |
|---|---|---|
| Split per-vault config into `purpose.md` (the *why*) and `schema.md` (the *how / taxonomy*) | Cleaner pattern-vs-domain separation; matches downstream convention; allows the skill to ship the workflow once instead of duplicating it in every vault's CLAUDE.md | `nashsu/llm_wiki` |
| **5 scenario templates** (research / reading / personal / business / general) covering Karpathy's enumerated use cases | Eliminates the "what page types should I have" decision for new users; per-scenario `purpose.md` + `schema.md` defaults | `nashsu/llm_wiki` |
| **Free-form context** as third arg to `/llm-wiki:init` | Lets users one-shot the bootstrap by describing the wiki in prose; the skill drafts `purpose.md` and `schema.md` from the description | this design |
| **Cross-vault citation syntax `[[wiki-name:page-slug]]`** | Required for cross-vault query results; the bare `[[page]]` form is ambiguous when multiple vaults are involved | this design |
| **Smart batch ingest** ("ingest all new sources" finds unprocessed files in `raw/`) | Reduces the cost of clearing a backlog; preserves the supervised pause per source | this design |
| **SHA-256 idempotency cache** at `.llm-wiki/ingest-cache.json` | Prevents wasted LLM calls on re-ingest of unchanged sources; tracks files-written for cascade-delete | `nashsu/llm_wiki` |
| **Optional qmd integration** with size-threshold fallback | Karpathy explicitly mentions qmd; we use it only above ~100 pages where index-scan struggles, otherwise let `index.md`-first navigation work | Karpathy gist |
| **Vault registry** at `~/.llm-wiki/vaults.json` | Required for cross-vault query to know what vaults exist; populated by `/llm-wiki:init` automatically | this design |
| **`synthesis.md` as a single top-level file** (rather than a folder) | Karpathy mentions a "synthesis" page type but doesn't fix the layout; a single evolving-position file is simpler than a sub-directory and matches the "one current take" intent | this design + our manual wiki |

### What we explicitly do NOT adopt from `nashsu/llm_wiki`

- ❌ Custom desktop / web UI (we keep Obsidian as the IDE — Karpathy's intent)
- ❌ Two-stage CoT ingest (single-stage is simpler; the LLM is capable enough)
- ❌ Knowledge graph with weighted edges + community detection (Obsidian's graph view is sufficient at our scale)
- ❌ Asynchronous review queue (synchronous "discuss-then-write" pause is enough)
- ❌ Built-in vector embedding (qmd handles this when needed; otherwise tokenized search via index.md is fine)
- ❌ Chrome web clipper of our own (use Obsidian Web Clipper)

The principle: **add a feature only if it serves Karpathy's pattern; reject if it replaces a piece of Karpathy's pattern (especially Obsidian).**

---

# Part 2 — Skill Design

## 2.1 Plugin Architecture

The skill ships as a Claude Code plugin. Local install path:

```
~/.claude/plugins/llm-wiki/
├── plugin.json                          ← plugin manifest (name, version, author, components)
├── marketplace.json                     ← marketplace listing metadata
├── README.md                            ← user-facing intro + install + quickstart
├── skills/
│   └── llm-wiki/
│       └── SKILL.md                     ← pattern guidance, page conventions, workflow descriptions
└── commands/
    ├── init.md                          ← /llm-wiki:init slash command
    ├── ingest.md                        ← /llm-wiki:ingest slash command
    ├── query.md                         ← /llm-wiki:query slash command
    ├── research.md                      ← /llm-wiki:research slash command
    ├── lint.md                          ← /llm-wiki:lint slash command
    ├── list.md                          ← /llm-wiki:list (small helper, prints vault registry)
    └── rm.md                            ← /llm-wiki:rm (safe vault deletion)
```

### Why this structure

- **One skill** rather than multiple (`llm-wiki-ingest`, `llm-wiki-query`, etc.) because the workflows reference each other and read better as one document.
- **Slash commands** for explicit invocation of each operation. They each invoke the skill (so the workflow is loaded), then run the operation-specific prompt. The plugin ships **1 skill + 7 slash commands**.
- **Both invocation paths converge** — `/llm-wiki:ingest <url>` and "ingest this url" both end up running the same workflow because the skill auto-loads via description match in either case.

### Marketplace

`marketplace.json` makes the plugin installable via `claude plugins install <repo-or-name>`. v1 ships with the plugin functional but not yet listed in any public marketplace; user installs locally by cloning into `~/.claude/plugins/`. Promoting to a public marketplace is an "add later" step, not v1 work.

## 2.2 The Skill (`SKILL.md`)

The skill file is the **pattern's brain**. It encodes everything that's the same across every wiki:

### Front-matter (auto-trigger)

```yaml
---
name: llm-wiki
description: |
  Use when the user wants to bootstrap, maintain, query, or lint an llm-wiki vault — a personal
  knowledge base where the LLM continuously builds and maintains structured Markdown pages from
  source documents (PDFs, papers, articles, web clips, notes). Triggers on phrases like "ingest
  this paper", "what does the wiki say about X", "lint the wiki", "create a new wiki for X",
  or when the user is in a directory containing purpose.md.
---
```

The description matters — it determines auto-trigger reliability when users phrase intent naturally.

### Content (what's in `SKILL.md`)

1. **What this skill assumes** — you're working with a *vault* directory containing `purpose.md`, `schema.md`, `raw/`, and `wiki/`. If those aren't present and the user asks for an op other than `/llm-wiki:init`, the skill politely refuses and points at `/llm-wiki:init`.
2. **Vault layout reference** (full ASCII tree, see §2.4)
3. **Page-conventions reference** (see §2.5)
4. **Workflow procedures**:
   - **Ingest workflow** (full procedure, see §2.3.2)
   - **Query workflow** (full procedure, see §2.3.3)
   - **Lint workflow** (full procedure, see §2.3.4)
5. **Default page types and frontmatter** (see §2.5)
6. **Cross-vault behavior** — how citation syntax works, how the registry is read (see §2.6)
7. **qmd usage** — when to invoke, fallback (see §2.8)
8. **Extraction tooling** — how to convert non-Markdown sources (see §2.9)

The `SKILL.md` is essentially the same content as our reference `~/git/llm-world-models-wiki/CLAUDE.md`, with the *domain-specific* parts removed (since those live in each vault's `purpose.md` + `schema.md`).

## 2.3 Slash Commands

The plugin ships 7 slash commands. Each is a thin trigger that invokes the skill and runs an operation-specific prompt.

### 2.3.1 `/llm-wiki:init <name> [<scenario>] ["<free-form context>"]`

Bootstrap a new wiki vault.

**Args:**
- `<name>` (required) — directory name or path. **Bare** name → resolved as `<cwd>/<name>` and the user is prompted to confirm or redirect (added v0.4.1). **Explicit path** (starts with `/` or `~`) → used as-is, no prompt.
- `<scenario>` (optional, default `research`) — one of: `research | reading | personal | business | general`
- `"<free-form context>"` (optional) — a sentence or paragraph describing the wiki's domain, headline question, scope. If provided, the skill drafts a richer `purpose.md` and seeds branches in `schema.md`; if absent, both files get scenario-templated stubs with section headers and prompts.

**Workflow:**

1. Validate `<scenario>` is one of the five known scenarios. Default to `research`.
1.5. **Path-confirmation prompt (bare names only, v0.4.1):** If `<name>` was bare, show the resolved path and ask `Create vault here? (y / n / different absolute-or-~ path)`. Treat empty/`y` as accept; `n` as abort; any other reply must start with `/` or `~` (else abort with a typed-message error). Skip this step if `<name>` was already an explicit path — the user has already chosen a location.
2. Refuse if the resolved directory already exists.
3. Scaffold the standard tree (raw/sources, raw/clips, raw/assets, wiki/{entities,concepts,sources,comparisons,questions}).
4. **If free-form context provided**:
   - LLM call: draft `purpose.md` (domain, headline question, scope-in/out, initial thesis) and `schema.md` (4–6 starter branches inferred from the domain) following the chosen scenario's structure.
   - Save both files.
   - Show drafts back to user, suggest review/refine.
5. **If no free-form context**: write scenario-specific templated stubs for `purpose.md` and `schema.md` with section headers + prompts (e.g., "## Headline question\n\n*<your question here>*").
6. Create `wiki/index.md` (skeleton with section headers per page type), `wiki/log.md` (with init entry), `wiki/synthesis.md` (empty stub).
7. Run `git init`, write `.gitignore` (excludes `.obsidian/workspace*`, `.DS_Store`, `__pycache__/`, etc.).
8. Append entry to `~/.llm-wiki/vaults.json` (creating the registry if it doesn't exist).
9. Print one-screen summary: where the vault lives, what scenario was applied, suggested next steps (open in Obsidian; configure Web Clipper to write to `raw/clips/`; ingest first source).

**Scenarios:**

| Scenario | Domain examples | Default page types | Notes for `purpose.md` template |
|---|---|---|---|
| **research** ★ default | "deep dive on transformers", "do LLMs have world models", "competitive analysis of X" | entity, concept, source, comparison, question, synthesis | Headline question, scope-in/out, evolving thesis |
| **reading** | "Frankenstein 1818 vs 1831", "all of Calvino in 2026" | character, theme, chapter, plot-thread, quote, source | Book(s) being read, themes/threads to track, current reading position |
| **personal** | "self-improvement journal", "fitness + nutrition tracking" | goal, journal-entry, person, habit, insight, source | Goals, life areas tracked, journaling cadence |
| **business** | "team OKR retrospective", "customer-research backlog" | customer, project, decision, retrospective, person, source | Team / project / customer focus, decision-log emphasis |
| **general** | trip planning, hobby deep-dive | note, source, person, concept | Open-ended ("what are you trying to learn?") |

All scenarios share `index.md`, `log.md`, `synthesis.md` — only `purpose.md` and `schema.md` defaults differ.

### 2.3.2 `/llm-wiki:ingest <url-or-path> [--scaffold] [--re-ingest] [--all]`

Ingest a source into the current vault.

**Args:**
- `<url-or-path>` (optional if `--all`) — a URL, an absolute path, or a path relative to the vault root. If absent, skill asks user.
- `--scaffold` — treat the source as a *field-map* (e.g. survey or awesome-list); seed many entity/concept/question stubs but do NOT file the source as a thesis claim
- `--re-ingest` — force re-processing even if the SHA-256 hash matches the cache
- `--all` — smart batch: scan `raw/sources/` and `raw/clips/`, find files not in cache, present list, ingest all (with the supervised pause per source)

**Mode:** if `--scaffold` is set, the workflow runs in *field-map mode*: the source page is filed as a field-map (not a thesis source); entity/concept/question pages are *stubs* (one paragraph each, no rich content); the log entry uses the `scaffold` prefix instead of `ingest`. Otherwise the workflow runs in *standard mode* (full source page, rich entity/concept pages, normal log entry).

**Workflow (single source):**

1. **Resolve the source.**
   - URL → fetch via [firecrawl](https://github.com/mendableai/firecrawl-cli) skill (the user's global default for web). If firecrawl unavailable, fall back to WebFetch (HTML) or `curl`+Read (PDFs). Save extracted markdown to `raw/clips/<slug>.md`. Slug = filename minus extension, kebab-cased.
   - Path inside vault — leave in place.
   - Path outside vault — copy into `raw/sources/`.
2. **Compute SHA-256 of source content. Check `.llm-wiki/ingest-cache.json`.**
   - If hash matches and `--re-ingest` not set: print "already ingested, hash unchanged; pass `--re-ingest` to force" and exit.
   - Otherwise proceed.
3. **Extract to markdown if needed** (see §2.9). Output staged at `raw/clips/<slug>.md`.
4. **Read source. Discuss with user.** Summarize key takeaways in 1–2 paragraphs; ask for emphasis direction. **Pause and wait** — never silently barrel through. (Karpathy's "supervised" pattern.)
5. **After user response, write wiki pages** (content depth depends on mode):
   - Write `wiki/sources/<slug>.md` (standard: citation + key claims + evidence quality + branch tag; scaffold: field-map summary describing what the source surveys/covers).
   - Create or merge `wiki/entities/`, `wiki/concepts/`, `wiki/comparisons/`, `wiki/questions/` pages — typically 5–15 pages touched (standard: rich content; scaffold: one-paragraph stubs only).
   - Update `wiki/index.md` to list the new source page + any new entity/concept stubs.
   - Update `wiki/synthesis.md` *only if* the source changes the cross-cutting position (standard mode only — scaffold mode never updates synthesis). Be explicit about what changed and why.
   - Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | <title>` (standard) or `## [YYYY-MM-DD] scaffold | <title>` (scaffold), with 3–5 bullets.
5.5. **Purpose drift detection** (standard mode only; skipped in scaffold mode):
   - Check `<vault>/.llm-wiki/drift-skip.txt` — if the source slug appears with `skip-drift`, skip silently.
   - Compare the newly written source page's `tags` and `key_claims` against `purpose.md`'s `## Domain`, `## Scope — in`, and `## Scope — out` sections (or scenario-equivalent sections for non-research vaults, e.g. `## Book(s)`, `## Life areas`).
   - Heuristic fires **only on clear new sub-areas or explicit scope-out violations**. Borderline cases pass silently (lean toward no prompt).
   - If drift is detected, show a proposed update to `purpose.md` and prompt: `"This source expands the wiki's scope. Update purpose.md? (y/n/s)"`:
     - **(y) Save**: writes the proposed update and continues.
     - **(n) Skip this time**: continues without saving.
     - **(s) Always skip for this source**: appends `<source-slug>: skip-drift` to `.llm-wiki/drift-skip.txt` and continues.
   - The proposed update is **always shown** before saving — never auto-saved silently.
   - Scenario-aware: matches section names to the vault's chosen scenario template.
6. **Update SHA-256 cache** with files-written list.
7. **If qmd present:** run `qmd index --update <vault>` (cheap when warm).

**Workflow (batch via `--all` or natural language "ingest all new sources"):**

1. Walk `raw/sources/` and `raw/clips/`, list every file.
2. Cross-reference `.llm-wiki/ingest-cache.json` — find files whose hash isn't cached or differs from cached value.
3. Print: `"Found N unprocessed sources: <list>. Ingest all in order?"`
4. **Always ask first** — wait for user confirmation. User can say "yes", "skip B", "do A and D only", etc.
5. For each, run the standard supervised single-source workflow above. The discussion-pause stays in place; user can move quickly ("emphasis: same, proceed") or pause to direct.

### 2.3.3 `/llm-wiki:query <question> [--all | --vaults a,b]`

Answer a question against the wiki.

**Args:**
- `<question>` (required) — the question to answer (quote if multi-word; or use natural language)
- `--all` — query across every vault listed in `~/.llm-wiki/vaults.json`
- `--vaults a,b,c` — query across the named vaults only (comma-separated)

If neither `--all` nor `--vaults` is provided, query operates on the cwd-detected vault.

**Workflow (single vault):**

1. Read `wiki/index.md` first.
2. **If qmd present and wiki ≥ ~100 pages:** `qmd search "<question>"` + rerank top-K (default 8).
3. **Otherwise:** select relevant pages by index scan + filename match (typical: 5–10 pages).
4. Read those pages. Read `wiki/synthesis.md` and `purpose.md` for cross-cutting context.
5. Answer with `[[wiki-page]]` citations that bottom out at `sources/` pages.
6. **If the answer is non-trivial** (a comparison, novel synthesis, new framing): offer to file as `wiki/comparisons/<slug>.md`, `wiki/questions/<slug>.md`, or as an entry in `wiki/synthesis.md`. Suggest the page type based on shape; ask before filing.
7. Append `## [YYYY-MM-DD] query | <question>` to `log.md` with one bullet on the answer + where it landed (if filed back).

**Workflow (cross-vault via `--all` / `--vaults a,b`):**

1. Read `~/.llm-wiki/vaults.json`. Validate target vaults exist on disk.
2. For each target vault: read its `purpose.md`, `synthesis.md`, `index.md`, then run the per-vault query (same logic as above; qmd if available).
3. Compose a multi-section answer:
   - **Per vault**: what each wiki contributes, with `[[wiki-name:page-slug]]` citations.
   - **Cross-cutting synthesis**: the meta-answer that spans the vaults, with disagreements/agreements between wikis surfaced explicitly.
4. **Filing back is opt-in and explicit**: "This cross-cutting synthesis seems valuable. File to (a) wiki-A's synthesis, (b) wiki-B's synthesis, (c) save without filing, (d) discard." Cross-vault answers do *not* silently land in any single wiki's `synthesis.md`.
5. Append `## [YYYY-MM-DD] cross-query | <question>` to *each queried vault's* `log.md` with one bullet per vault on what was contributed.

**Citation syntax across vaults:** `[[wiki-name:page-slug]]`. Within a single vault, the bare `[[page-slug]]` form is preserved.

### 2.3.4 `/llm-wiki:lint`

Health-check the current vault.

**Workflow:**

1. Walk every page in `wiki/` and parse frontmatter.
2. Run the following checks:

| # | Check | What it detects | Severity | Auto-fixable |
|---|---|---|---|---|
| 1 | Orphan `[[wikilinks]]` | Links pointing to non-existent pages → list missing slugs | Error | Yes (remove the `[[]]`) |
| 2 | Dangling pages | Pages with no inbound `[[wikilinks]]` AND not in `index.md` | Warning | No |
| 3 | Index drift | Pages that exist in `wiki/` but aren't listed in `index.md` (and vice-versa) | Warning | Yes (add/remove index entries) |
| 4 | Stale claims | Source pages with claims contradicted by newer source pages (heuristic, LLM-judged on a small sample) | Warning | No |
| 5 | Frontmatter validity | Missing required fields (`type`, `title`, `created`, `updated`, `sources`) | Error | Yes (stub empty fields) |
| 6 | Schema conformance | `branch:` values not in `schema.md`'s declared branch list | Warning | No |
| 7 | Synthesis lag | `synthesis.md`'s `updated:` field older than newest source page | Warning | No |
| 8 | Missing concept pages | Concepts mentioned in 3+ pages without their own page (heuristic) | Suggestion | No |
| 9 | Purpose drift | ~30% of source pages diverging from the stated scope, or any source explicitly matching the stated scope-out section | Suggestion | No |

3. Produce a markdown report grouped by severity (error / warning / suggestion).
4. Ask: "Found N issues. Want me to (a) fix all auto-fixable, (b) walk through each and decide, (c) just save the report and we'll do it later?"
5. **Auto-fixable:** orphan-link cleanup (remove the `[[]]`), frontmatter field stubbing (add empty `tags: []` etc.), index updates (add missing entries / remove deleted entries).
6. **Not auto-fixable** (requires user judgment): stale-claim resolution, missing-concept-page authoring, purpose-drift resolution (decide whether to update `purpose.md` or relocate sources to a different vault).
7. Append `## [YYYY-MM-DD] lint | N issues` to `log.md` with the count summary.

Lint is **per-vault only** (operates on cwd or `--vault <path>`). Cross-vault analogues are deferred — see Part 4.

### 2.3.5 `/llm-wiki:list`

A small helper that prints the contents of `~/.llm-wiki/vaults.json` in a readable format. Useful when running `/llm-wiki:query --vaults ...` and you've forgotten the registered names.

### 2.3.6 `/llm-wiki:research "<topic>" [--max-sources N] [--unsupervised] [--vault <path>]`

Search the web for a topic, triage candidates against the vault's `purpose.md`, and ingest user-selected sources via the existing ingest pipeline.

**Args:**
- `"<topic>"` (required) — the research topic, quoted if multi-word
- `--max-sources N` (default 20) — cap on candidates returned by web search
- `--unsupervised` — skip per-source discussion pauses after the initial curated-list confirmation; use when batching through known-good sources
- `--vault <path>` — override cwd-detected vault

**Workflow:**

0. **Detect vault.** Walk up from cwd to find `purpose.md`, or use `--vault`. Refuse with "Not a vault" error if not found. Read `purpose.md` for relevance grounding.
0.5. **purpose.md filled-state check.** Scan `purpose.md` for italicized placeholder pattern `\*<[^>]+>\*` (e.g. `*<the topic / area being studied>*`). If **3 or more** placeholders are found, the file is still a template stub; prompt the user:
   - **(y) Draft now.** LLM generates a filled `purpose.md` from the research topic, using the same free-form-context drafting logic as `/llm-wiki:init <name> <scenario> "<context>"`. Scenario is inferred from the existing template's section structure. The draft is shown for confirmation before saving; the user may also choose to edit before saving. Once saved, triage proceeds with a properly grounded `purpose.md`.
   - **(n) Proceed with weak triage.** Warning is printed and Step 1 runs with degraded relevance scoring.
   - **(e) Edit manually.** Vault path is printed and command exits cleanly. User fills `purpose.md` themselves and re-runs.
   - If fewer than 3 placeholders are found, Step 0.5 is a no-op.
1. **Web search.** Use the firecrawl skill (the user's preferred default per global instructions) with `--limit <max-sources>`. Falls back to WebSearch if firecrawl unavailable. Parse `{title, url, snippet}` per candidate.
2. **LLM triage.** For each candidate, score against `purpose.md`:
   - **Relevance** (1–5★): match against the headline question and scope-in.
   - **Type tag**: `paper` | `blog` | `news` | `talk` | `spec` | `other`. arXiv URLs and known academic venues default to ★★★★ unless clearly off-topic.
   - **Already-covered check**: cross-reference URL/title slug against `wiki/sources/*.md`.
3. **Present + ask.** Display sorted-by-relevance list with stars, type, snippet, already-covered marker. Selection syntax: `1,3,5` / `1-5` / `all` / `all stars 4+` / `skip blogs` / `none`. **Always ask first** — Karpathy-faithful supervised pattern.
4. **Per chosen source.** Fetch (firecrawl scrape) → save to `<vault>/raw/clips/<slug>.md` → run the standard ingest workflow (§2.3.2 single-source steps 1–7, including Step 5.5 drift detection). Per-source discussion pause is included unless `--unsupervised`. Cache hits (already-covered) skipped silently.
5. **Summary log entry.** *In addition to* the per-source `ingest` entries from step 4, append one summary entry:
   ```
   ## [YYYY-MM-DD] research | <topic>
   - Searched web for "<topic>" → N candidates
   - User selected M sources to ingest: <short-titles>
   - Skipped K already-covered, R user-rejected
   ```

**Edge cases:**
- **Vault has unfilled `purpose.md`**: handled by Step 0.5 above.
- **Web search returns 0 results**: report and suggest broadening the query.
- **All candidates already-covered**: report; suggest `/llm-wiki:query "<topic>"` to surface what's already known.
- **One source fetch fails**: continue with the rest, summarize failures at the end.
- **User selects "none"** or aborts: do nothing, no log entry.

### 2.3.7 `/llm-wiki:rm <name-or-path>`

Safely delete a vault and de-register it from `~/.llm-wiki/vaults.json`.

**Args:**
- `<name-or-path>` (required) — the registered vault name (from `~/.llm-wiki/vaults.json`) or an absolute path to the vault directory.

**Workflow:**

1. **Resolve path.** If a name is given, look it up in `~/.llm-wiki/vaults.json`. Reject if not found; suggest `/llm-wiki:list` to see registered vaults.
2. **Hard safety checks (unconditional refusal — cannot be overridden):** refuse outright if the resolved path is `/`, `""`, `$HOME`, `/Users/$USER`, or `/Users`. These checks fire before any user confirmation.
3. **Soft sanity check:** if `purpose.md` is absent at the resolved path, warn "No purpose.md found at <path> — this may not be an llm-wiki vault" and ask `Continue anyway? (y/n)`. Protects against mistyped paths.
4. **Confirmation prompt.** Show the resolved name + path + registry entry + disk size. Ask `"Delete vault '<name>' at <path>? This is irreversible. (yes/no)"` — require typing "yes" (not just "y").
5. **Delete.** Run `rm -rf <path>`. If the directory is already gone, skip `rm -rf` and continue to the registry update.
6. **Registry update.** Write the updated `vaults.json` atomically: write to `.json.tmp`, then `mv` over the original.
7. **Confirm.** Print `"Vault '<name>' deleted and de-registered."` The printed output is the only audit trail — there is no vault to write a log entry to.

**Design constraints:**
- No log entry is created (the vault is gone).
- The command is intentionally narrow — it does not cascade-delete pages from other vaults that cited this vault (cross-vault citation cleanup is manual).
- Registry write-path is now symmetric: `/llm-wiki:init` creates entries; `/llm-wiki:rm` removes them.

## 2.4 Vault Files

```
<vault>/                                ← anywhere on disk; cwd-detected by purpose.md presence
├── purpose.md                          ← per-domain — required
├── schema.md                           ← per-domain — optional (skill defaults if absent)
├── raw/                                ← immutable
│   ├── sources/                        ← user-curated PDFs / DOCX / notes
│   ├── clips/                          ← Obsidian Web Clipper output + ingest-time markdown extractions
│   └── assets/                         ← inline images (Obsidian attachment folder target)
├── wiki/                               ← LLM-maintained
│   ├── index.md                        ← content catalog (skill maintains)
│   ├── log.md                          ← chronological log (skill appends)
│   ├── synthesis.md                    ← evolving cross-cutting position (skill maintains)
│   ├── entities/                       ← people, models, organizations
│   ├── concepts/                       ← theories, methods, abstractions
│   ├── sources/                        ← per-source summary pages
│   ├── comparisons/                    ← side-by-side analyses
│   └── questions/                      ← open questions with evolving answers
├── .obsidian/                          ← Obsidian creates on first vault-open (NOT skill)
├── .llm-wiki/                          ← skill sidecar, hidden, rebuildable
│   ├── ingest-cache.json               ← SHA-256 idempotency cache
│   └── drift-skip.txt                  ← created on demand; records sources where drift detection is permanently skipped
└── .git/                               ← skill runs `git init` during /llm-wiki:init
```

### 2.4.1 `purpose.md` (per-domain, required)

The wiki's *why*. The skill reads it on every ingest and query for context. Drafted by `/llm-wiki:init` (templated or LLM-drafted from free-form context); edited by the user; the skill may *suggest* updates during ingest but doesn't silently rewrite.

**Standard sections** (per scenario; example shown for `research`):

```markdown
# Purpose

## Domain
[The topic / area being studied]

## Headline question
[The one question this wiki is meant to help answer]

## Scope — in
[What's included]

## Scope — out
[What's deliberately excluded — scope-control discipline]

## Evolving thesis
[Current best understanding; revised as the wiki grows. Empty / "no thesis yet" is fine for a fresh wiki.]

## Sources strategy
[Where new sources come from — arxiv subscriptions, podcast feeds, advisor recommendations, etc.]
```

Other scenarios have analogous sections (e.g. `reading` swaps "Domain" for "Book(s)" and "Headline question" for "Themes tracking").

### 2.4.2 `schema.md` (per-domain, optional)

The wiki's *taxonomy*. Defines branches and any page-type overrides. Optional — if absent, the skill uses scenario defaults.

```markdown
# Schema

## Branches
- branch-name-1 — short description
- branch-name-2 — short description
- ...

## Page types
[Inherits scenario defaults from skill. Override only if you need a custom type.]

## Custom rules
[Any vault-specific conventions beyond the skill defaults — frontmatter additions, citation styles, etc.]
```

Concrete example for transformer-wiki:

```markdown
# Schema

## Branches
- architecture — model design, attention, FFN, normalization
- training — optimization, data, scaling laws
- evaluation — benchmarks, behavior probes
- applications — NLP, vision, audio, multimodal
- history — chronology and lineage (Vaswani 2017 → BERT → GPT → ...)
- cross — cross-cutting

## Page types
(Uses default page types from llm-wiki skill: entity, concept, source, comparison, question, synthesis.)

## Custom rules
- Source pages additionally include `arxiv_id:` field where applicable.
```

### 2.4.3 `wiki/index.md` (skill-maintained)

Content catalog. Updated on every ingest and lint. Initial skeleton from `/llm-wiki:init`:

```markdown
# Index

## Entities
*(none yet)*

## Concepts
*(none yet)*

## Sources
*(none yet)*

## Comparisons
*(none yet)*

## Questions
*(none yet)*

## Synthesis
[[synthesis]] — current cross-cutting position
```

After ingest, each section gets entries like:

```markdown
## Entities
- [[othello-gpt]] — Li et al. 2022, methodological progenitor of probe+intervention work
- [[transformer-architecture]] — Vaswani 2017
```

The skill's first action on `/llm-wiki:query` is to read this file.

### 2.4.4 `wiki/log.md` (skill-maintained, append-only)

Chronological audit trail. Each operation appends one section. Format mandated by Karpathy's gist:

```
## [YYYY-MM-DD] <type> | <title>
- bullet
- bullet
```

Where `<type>` is one of `init | ingest | scaffold | re-ingest | query | cross-query | research | lint`.

| Type | When emitted |
|---|---|
| `init` | `/llm-wiki:init` creates a new vault |
| `ingest` | `/llm-wiki:ingest` adds a source in standard mode |
| `scaffold` | `/llm-wiki:ingest --scaffold` files a source as field-map |
| `re-ingest` | `/llm-wiki:ingest --re-ingest` re-processes an existing source |
| `query` | `/llm-wiki:query` (single-vault) |
| `cross-query` | `/llm-wiki:query --all` or `--vaults` |
| `research` | `/llm-wiki:research` summary entry (per-source ingests also emit `ingest`) |
| `lint` | `/llm-wiki:lint` |

Note: `/llm-wiki:rm` does **not** emit a log type — the vault is deleted and there is nowhere to write to.

Greppable: `grep "^## \[" wiki/log.md | tail -10` shows recent activity.

### 2.4.5 `wiki/synthesis.md` (skill-maintained)

Single evolving cross-cutting position. Has frontmatter (type=synthesis). Updated *only when an ingest changes the position* — explicitly. Written by the skill, but the user is welcome to edit.

### 2.4.6 `.llm-wiki/ingest-cache.json` (skill-managed)

Format:

```json
{
  "raw/sources/vaswani-2017-attention.pdf": {
    "sha256": "127c8dcb...",
    "last_ingested": "2026-05-06T14:03:12Z",
    "files_written": [
      "wiki/sources/vaswani-2017-attention.md",
      "wiki/entities/transformer-architecture.md",
      "wiki/concepts/scaled-dot-product-attention.md",
      "wiki/concepts/multi-head-attention.md"
    ]
  }
}
```

Keyed by source path (relative to vault root). The `files_written` list enables cascade-delete: when a source is removed, the skill knows which wiki pages it contributed to.

## 2.5 Page Types and Frontmatter

### Default page types (shipped in skill)

| Type | Purpose | Lives in |
|---|---|---|
| `entity` | A specific named thing — a model, person, lab, dataset, product | `wiki/entities/` |
| `concept` | An abstract idea, technique, theory, phenomenon | `wiki/concepts/` |
| `source` | A summary of one ingested document | `wiki/sources/` |
| `comparison` | Explicit side-by-side analysis of 2+ entities or concepts | `wiki/comparisons/` |
| `question` | An open question, with the wiki's evolving best answer | `wiki/questions/` |
| `synthesis` | The cross-cutting position file (`synthesis.md`) | `wiki/` (top-level file) |

Scenarios can override or extend this set (see §2.3.1 scenarios table). For example, the `reading` scenario adds `character`, `theme`, `chapter`, `plot-thread`, `quote`.

### Required frontmatter

```yaml
---
type: entity | concept | source | comparison | question | synthesis | <scenario-specific>
title: <human-readable title>
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: [list of optional tags]
related: [list of slug refs]
sources: [list of source slugs that contributed to this page]
---
```

`branch:` field is required IF the vault's `schema.md` declares branches; absent otherwise.

### Source-specific additions

For `type: source` pages, additional fields:

```yaml
authors: [...]
year: YYYY
venue: <publication / website>
url: <canonical URL>
key_claims: ["...", "..."]
```

### Cross-references

- Within a vault: `[[page-slug]]` (Obsidian-compatible)
- Across vaults: `[[wiki-name:page-slug]]` (our addition)
- Frontmatter `related:` and `sources:` arrays use slug strings (e.g. `["transformer-architecture", "scaled-dot-product-attention"]`)

## 2.6 Cross-Vault — Registry and Behavior

### 2.6.1 Registry file

```
~/.llm-wiki/vaults.json
```

```json
{
  "vaults": [
    {
      "name": "transformer-wiki",
      "path": "/Users/f.luo/git/transformer-wiki",
      "added": "2026-05-06",
      "purpose_excerpt": "Deep dive on transformer architectures, attention mechanisms..."
    },
    {
      "name": "llm-world-models-wiki",
      "path": "/Users/f.luo/git/llm-world-models-wiki",
      "added": "2026-04-30",
      "purpose_excerpt": "Personal research wiki on world models in the LLM domain..."
    }
  ]
}
```

### 2.6.2 Population

- `/llm-wiki:init` auto-appends an entry for the new vault.
- `/llm-wiki:rm` removes the entry when a vault is deleted. Registry writes are atomic (write to `.json.tmp`, then `mv` over the original).
- User can hand-edit the file.
- `/llm-wiki:list` (helper command) prints registered vaults.
- If a registered path no longer exists on disk, cross-vault queries silently skip it (with a warning); the entry stays unless user removes it or uses `/llm-wiki:rm`.

### 2.6.3 What single-vault commands ignore

`/llm-wiki:init`, `/llm-wiki:ingest`, `/llm-wiki:lint`, and `/llm-wiki:rm` never read the registry for routing. They operate on the cwd-detected vault (or an explicit name/path argument). `/llm-wiki:rm` *writes* to the registry to remove the deleted vault's entry.

### 2.6.4 What only `/llm-wiki:query --all` and `/llm-wiki:query --vaults` use

Just those two flags read the registry. The cross-vault workflow is fully described in §2.3.3.

## 2.7 SHA-256 Ingest Cache

Already specified in §2.4.6. Behavior summary:

- Compute hash on source content (not source path).
- **Cache key** = vault-relative path of the source *after resolution* (i.e. after a URL has been fetched and saved to `raw/clips/<slug>.md`, or an external file copied into `raw/sources/`). Cache keys never reference paths outside the vault. URLs themselves aren't cache keys — the resolved local file is.
- Skip ingest if hash matches; print message and exit.
- Update cache after successful ingest.
- `--re-ingest` ignores the cache.
- The `files_written` list is consulted on cascade-delete.

## 2.8 qmd Integration

### Detection

Skill checks `which qmd` at startup of `/llm-wiki:query`. If absent, skips silently and uses index-scan navigation.

### When invoked

| Wiki size | qmd installed | Query strategy |
|---|---|---|
| Any | ✗ | Read `index.md`, then drill into matched pages |
| < 100 pages | ✓ | Same as above (qmd is overkill) |
| ≥ 100 pages | ✓ | `qmd search "<question>" --top 8` + rerank, then read top hits |

The 100-page threshold matches Karpathy's gist claim.

### Reindex

After every successful `/llm-wiki:ingest`, if qmd present, run `qmd index --update <vault>`. Cheap when warm.

After `/llm-wiki:lint`'s auto-fix (which may delete pages), same — `qmd index --update`.

### v1: CLI only, no MCP

The skill calls qmd via the Bash tool. qmd's MCP server is not wired into the plugin in v1. We document MCP wiring as an opt-in v1.5 add for users who want lower-latency tool access.

### Install

The skill does not install qmd. `/llm-wiki:init`'s post-install message says: *"Optional: install qmd (https://github.com/tobi/qmd) for fast hybrid search once your wiki passes ~100 pages."*

## 2.9 Document Extraction

When `/llm-wiki:ingest` is given a non-Markdown source, the skill extracts to Markdown before reading. Format priorities:

| Format | First try | Fallback | External dep |
|---|---|---|---|
| `.md`, `.txt` | Read directly | — | none |
| `.pdf` | Claude Code's Read tool (handles PDFs natively) | `pdftotext <file> -` | poppler (only for fallback) |
| `.docx`, `.odt`, `.rtf`, `.epub` | `pandoc <file> -t markdown` | — | **pandoc** |
| `.pptx` | `pandoc -t markdown` | unzip + grep slide XMLs | pandoc |
| `.xlsx`, `.csv` | `python3 -c "import openpyxl; ..."` one-liner | `xlsx2csv` | `python3` (use `python3` invocation explicitly — `python` is unreliable cross-platform) |
| `.html` (URL) | firecrawl skill if available, else WebFetch | `curl` + `pandoc` | — |
| `.html` (file on disk) | `pandoc` | — | pandoc |

The skill's `/llm-wiki:init` post-install message warns: *"For DOCX/PPTX/RTF support, install pandoc: `brew install pandoc`."*

If the user attempts to ingest a format requiring a missing dependency, the skill says so and offers a suggestion (e.g. "install pandoc, or pre-convert this DOCX manually and drop the .md into raw/sources/").

Output of every extraction lands in `raw/clips/<slug>.md` (URLs and converted files alike). Originals stay where they are; the converted markdown is what the wiki workflow reads.

## 2.10 Distribution

### Plugin manifest (`plugin.json`)

```json
{
  "name": "llm-wiki",
  "version": "0.4.0",
  "description": "Karpathy's LLM Wiki pattern, as a Claude Code plugin. Bootstrap, maintain, query, research, and lint plain-markdown personal knowledge bases.",
  "author": {"name": "yellowstonely"},
  "components": {
    "skills": ["skills/llm-wiki/SKILL.md"],
    "commands": [
      "commands/init.md",
      "commands/ingest.md",
      "commands/query.md",
      "commands/research.md",
      "commands/lint.md",
      "commands/list.md",
      "commands/rm.md"
    ]
  }
}
```

### Marketplace listing (`marketplace.json`)

```json
{
  "name": "llm-wiki",
  "displayName": "LLM Wiki",
  "description": "Build a personal Markdown wiki that Claude maintains for you.",
  "tags": ["knowledge-base", "research", "obsidian", "karpathy", "wiki"],
  "homepage": "https://github.com/<user>/llm-wiki",
  "license": "MIT"
}
```

### Install paths

- **Local install (v1)**: clone the plugin repo into `~/.claude/plugins/llm-wiki/`. Reload Claude Code; plugin auto-discovers and loads.
- **Marketplace install (post-v1)**: `claude plugins install llm-wiki` once submitted to a public registry.
- **Update**: `git pull` in `~/.claude/plugins/llm-wiki/`. No state migration needed (vaults live separately).

### What's portable across machines

- The plugin (one git clone)
- Each wiki vault (independently — they're plain-markdown directories)
- The vault registry (`~/.llm-wiki/vaults.json` — sync if you want cross-machine consistency, otherwise per-machine)

---

# Part 3 — Non-Goals / Deferred to Future Versions

Explicitly out of scope for v1, with reasoning:

| Feature | Reason for deferring | Likely future version |
|---|---|---|
| Background daemon / file watcher (auto-ingest on file drop) | Skills are request-response; no persistent process model in Claude Code; adding one is a different product | Not planned (would replace pattern) |
| MCP server packaging | Bash + filesystem tools cover all v1 needs; MCP is a latency optimization not a capability gate | v1.5 if qmd integration becomes hot |
| `/llm-wiki:cross-check` (cross-vault contradiction finder) | Requires meta-schema to compare across vaults with different `schema.md` rules; non-trivial design | v1.5+ |
| Auto-vault discovery | Filesystem scan is fragile; explicit registry is honest | Not planned |
| Knowledge graph with weighted edges + community detection | Obsidian's graph view is sufficient at the user-scale we target; nashsu's 4-signal model is impressive but its absence isn't blocking | Not planned (use Obsidian) |
| Vector embedding search built into the skill | qmd does this when needed; reproducing it in the skill would duplicate effort | Not planned (use qmd) |
| Async review queue | Synchronous "discuss then write" pause is enough at our scale; nashsu's queue is for autonomous batch ingest, which Karpathy's pattern argues against | Not planned |
| Two-stage CoT ingest (analyze → generate) | Single-stage works for capable models; nashsu adopted this for output discipline with weaker models, less needed with Sonnet/Opus | Not planned |
| Web clipper of our own | Obsidian Web Clipper exists, configured once; no need to reproduce | Not planned |
| Hosted multi-tenant version | This is fundamentally a different product (lucasastorian's llmwiki.app pattern); skill is for personal use | Not planned |
| Autonomous research (no user selection step) | Violates the "human curates" principle of Karpathy's pattern | Not planned (would replace pattern) |
| Scheduled re-research (cron-style polling on a topic) | Skill is request-response; daemon is out of scope | v1.5+ if pain |
| Multi-engine search aggregation (Tavily + Brave + Bing simultaneous) | firecrawl already aggregates; multi-engine adds cost without clear quality win | v2 if firecrawl quality declines |
| Image / multimodal research candidates | All sources are text/markdown; vision sources require multimodal LLM ingest path | Out of scope for v1 |
| Auto-saving drift-detected `purpose.md` updates | `purpose.md` is owned by the user; the skill may suggest but never silently rewrite | Not planned |

---

# Part 4 — Acceptance Criteria

A v1 release is complete when:

### 4.1 Plugin layout
- [ ] `~/.claude/plugins/llm-wiki/` exists with `plugin.json`, `marketplace.json`, `README.md`
- [ ] `skills/llm-wiki/SKILL.md` is present and well-formed
- [ ] All 7 slash commands (`/llm-wiki:init`, `/llm-wiki:ingest`, `/llm-wiki:query`, `/llm-wiki:research`, `/llm-wiki:lint`, `/llm-wiki:list`, `/llm-wiki:rm`) exist as `commands/*.md`
- [ ] Plugin loads in Claude Code without errors

### 4.2 `/llm-wiki:init`
- [ ] `/llm-wiki:init test-wiki` creates a complete vault (raw/, wiki/, purpose.md, schema.md, all wiki/ skeleton files, `git init`-ed)
- [ ] All 5 scenarios produce scenario-appropriate `purpose.md` + `schema.md` templates
- [ ] Free-form context (`/llm-wiki:init x research "..."`) drafts richer purpose.md + seeded branches via LLM call
- [ ] Vault is registered in `~/.llm-wiki/vaults.json`
- [ ] Refuses to init if directory already exists

### 4.3 `/llm-wiki:ingest`
- [ ] Single-source ingest works for `.md`, `.txt`, `.pdf` (Read), `.docx` (pandoc), `.pptx` (pandoc), `.xlsx` (python)
- [ ] URL ingest works for HTML pages and arxiv PDFs (via curl + Read)
- [ ] Skill pauses after summarizing, asks for emphasis direction, waits for user
- [ ] Standard ingest produces source page + 5–15 entity/concept pages, updates index.md and log.md
- [ ] SHA-256 cache prevents redundant ingest of unchanged sources
- [ ] `--re-ingest` forces re-processing
- [ ] `--scaffold` produces field-map style stubs with log entry prefix `scaffold`
- [ ] `--all` finds unprocessed sources, asks before batch, processes serially with per-source pauses
- [ ] qmd reindex runs after ingest if qmd is on PATH

### 4.4 `/llm-wiki:query`
- [ ] Single-vault query reads index.md first, then relevant pages, produces answer with `[[wikilink]]` citations bottoming at sources/
- [ ] qmd is used when present and wiki ≥ 100 pages
- [ ] Index-scan fallback works without qmd
- [ ] Non-trivial answers prompt to file back as comparison/question/synthesis (asks before filing)
- [ ] `--all` reads vaults.json, queries every vault, produces multi-section answer with `[[wiki-name:page-slug]]` citations
- [ ] `--vaults a,b` queries specified subset
- [ ] log.md gets a `query` (or `cross-query`) entry per invocation

### 4.5 `/llm-wiki:lint`
- [ ] Detects orphan wikilinks, dangling pages, index drift, frontmatter validity, schema conformance
- [ ] Produces a markdown report grouped by severity
- [ ] Auto-fix mode handles orphan-link removal, frontmatter stubbing, index updates
- [ ] Manual-judgment items are flagged but not auto-resolved
- [ ] log.md gets a `lint` entry

### 4.6 `/llm-wiki:list`
- [ ] Prints registered vaults from `~/.llm-wiki/vaults.json` in readable format
- [ ] Marks vaults whose paths no longer exist on disk

### 4.7 Cross-platform / shell
- [ ] Plugin loads on macOS (primary target). Linux later (skill itself is markdown; should be portable, but pandoc/python/poppler must be available).
- [ ] No hard dependency on specific Claude Code version beyond what `plugin.json` declares.

### 4.8 Documentation
- [ ] `README.md` covers install, scenarios, all 7 slash commands with examples
- [ ] `SKILL.md` is self-contained: someone reading it cold understands the pattern + workflows
- [ ] Examples reference our two existing wikis (`~/git/transformer-wiki/`, `~/git/llm-world-models-wiki/`) where appropriate

### 4.9 `/llm-wiki:research` works
- [ ] Command appears in `/<tab>` autocomplete after plugin reload
- [ ] Refuses if cwd is not a vault and `--vault` not provided
- [ ] Web search returns results for a known topic (e.g. "scaling laws transformers")
- [ ] Triage produces star-rated, type-tagged candidates
- [ ] Already-covered detection works (re-running on the same topic with the same vault marks previously-ingested sources as already-covered)
- [ ] User selection syntax recognized: at minimum `all`, `1-3`, `none`
- [ ] Selected sources actually ingest (entity/concept pages created, index/log updated, cache populated)
- [ ] `--unsupervised` skips the per-source discussion pause; supervised default keeps it
- [ ] Summary log entry has format `## [date] research | <topic>` with the 3 expected bullets

### 4.10 `purpose.md` auto-draft offer fires on templated state
- [ ] Step 0.5 triggers when `purpose.md` has 3+ placeholder patterns matching `\*<[^>]+>\*`
- [ ] The three options (y/n/e) all behave correctly: `y` drafts and confirms before saving; `n` proceeds with weak-triage warning; `e` exits with vault path printed
- [ ] Fewer than 3 placeholders → no prompt (Step 0.5 is a no-op)
- [ ] Drafted `purpose.md` improves triage star ratings compared to the unfilled template

### 4.11 `/llm-wiki:rm` deletes vault + de-registers; safety checks fire
- [ ] Hard safety checks refuse deletion of `/`, `$HOME`, `/Users/$USER`, `/Users`, `""` unconditionally
- [ ] Soft sanity check warns and prompts when `purpose.md` is absent
- [ ] Full confirmation prompt shows name + path + size before proceeding
- [ ] `rm -rf` runs and vault directory is gone
- [ ] `~/.llm-wiki/vaults.json` no longer contains the deleted vault's entry
- [ ] If directory already missing, registry cleanup still happens
- [ ] No log entry is created anywhere

### 4.12 Drift detection prompts when source expands scope
- [ ] Step 5.5 fires when ingested source's claims clearly enter a scope-out area from `purpose.md`
- [ ] `y` saves the proposed `purpose.md` update; `n` skips; `s` writes to `drift-skip.txt` and skips
- [ ] A subsequent ingest of the same source slug is silently skipped if its slug is in `drift-skip.txt`
- [ ] Borderline cases pass silently (no false-positive prompts)

### 4.13 Purpose-drift lint check identifies divergence
- [ ] Check #9 fires when ~30% of source pages diverge from stated scope or any source matches the stated scope-out
- [ ] Severity is **Suggestion** (not Error or Warning)
- [ ] Check appears in lint report under "Suggestion" section
- [ ] Check is listed as not auto-fixable

---

# Part 5 — Open Risks

### 5.1 Skill description tuning

The auto-trigger description in `SKILL.md` determines whether natural-language phrasings reliably activate the skill. Too narrow ⇒ misses obvious cases ("ingest this paper" might not match if the description over-specifies). Too broad ⇒ over-triggers in unrelated contexts. **Mitigation:** start with explicit phrase examples in the description; iterate after using the plugin for a week.

### 5.2 Pandoc availability

`/llm-wiki:ingest`'s extraction tier depends on pandoc for DOCX/PPTX/RTF/EPUB. Users without pandoc see a warning. **Mitigation:** clear error message + link to install instructions; PDF/MD/TXT/HTML work without pandoc.

### 5.3 qmd availability

qmd is mentioned in Karpathy's gist but isn't a standard install. Different platforms have different package availability. **Mitigation:** purely optional; skill works without it; recommend in `/llm-wiki:init` post-install message.

### 5.4 LLM provider variability

Skill is provider-agnostic — works with whatever LLM Claude Code is configured with (Claude API, Claude CLI, OpenAI, Gemini, Ollama). But output quality of generated wiki pages depends on the model. Models prone to verbose/lazy output (smaller open models) may produce sparse pages. **Mitigation:** `SKILL.md` includes explicit instructions to produce thorough content; lint catches under-developed pages; user can configure their preferred model in Claude Code settings.

### 5.5 Cross-vault citation introduces new syntax

`[[wiki-name:page-slug]]` is *not* in Karpathy's gist. Some Markdown renderers and Obsidian itself may not recognize it as a valid wikilink. **Mitigation:** within Obsidian, the bare `[[page-slug]]` form continues to work for in-vault navigation; cross-vault citations are emitted only by `--all` / `--vaults` query results, which the user reviews in chat (not in Obsidian). If users want cross-vault citations to render in Obsidian, they can configure a custom plugin or treat cross-vault answers as live-in-chat artifacts.

### 5.6 Concurrent vaults

Two Claude Code sessions running `/llm-wiki:ingest` against the *same* vault simultaneously would race on `index.md`, `log.md`, and the cache file. Skill currently doesn't lock. **Mitigation:** document as a known limitation; add file-locking in v1.5 if pain emerges; for v1, "don't run two ingests at once on the same vault" is acceptable (matches Karpathy's "supervised, one at a time" recommendation).

### 5.7 Vault registry drift

If users move a vault directory without updating the registry, cross-vault queries may try paths that no longer exist. **Mitigation:** skill detects missing paths gracefully; `/llm-wiki:list` could highlight stale entries.

### 5.8 Web-search quality variability

firecrawl results depend on its underlying engine quality, which can drift. A topic that returned good arxiv-first results today may return SEO-spam blogs in 6 months. **Mitigation:** the LLM triage step filters by relevance and type, so star-ratings adapt to whatever firecrawl returns. If quality drops materially, users can pre-filter via `--max-sources 10` and lean on stars.

### 5.9 Token cost per research call

A research session is: 1 search call + up to 20 LLM relevance judgments + N source fetches + N×ingest pipelines. A heavy user running `/llm-wiki:research` daily could rack up real bills. **Mitigation:** `--max-sources` defaults to a reasonable 20; user controls how many sources actually ingest. Flagged in the README's "Optional integrations / cost considerations" section if pain emerges.

### 5.10 Replacing curation with delegation

Karpathy's pattern relies on the human picking authoritative sources. If `/llm-wiki:research` makes finding sources too easy, users may stop curating and the wiki quality declines toward LLM-popular-internet-content (blogs over papers). **Mitigation:** triage defaults bias toward papers (★★★★ for arxiv); the always-ask-first selection step keeps the human in the loop. Watch for telemetry / user feedback that wikis are accumulating low-quality sources.

---

## Appendix A — Quick reference for the skill author

A condensed cheat-sheet of the file inventory:

```
Plugin (lives once in ~/.claude/plugins/llm-wiki/):
  plugin.json
  marketplace.json
  README.md
  skills/llm-wiki/SKILL.md
  commands/init.md
  commands/ingest.md
  commands/query.md
  commands/research.md
  commands/lint.md
  commands/list.md
  commands/rm.md

Per-user (lives once in ~/.llm-wiki/):
  vaults.json

Per-vault (lives in each <vault>/):
  purpose.md (required)
  schema.md (optional)
  raw/sources/, raw/clips/, raw/assets/
  wiki/index.md, wiki/log.md, wiki/synthesis.md
  wiki/{entities,concepts,sources,comparisons,questions}/
  .llm-wiki/ingest-cache.json
  .llm-wiki/drift-skip.txt  (created on demand by /llm-wiki:ingest when user picks 's')
  .obsidian/ (created by Obsidian, not skill)
  .git/
```

## Appendix B — Glossary

- **Vault** — a directory containing `purpose.md` and a `wiki/` directory; the unit a single wiki occupies.
- **Source** — a raw input file (PDF, DOCX, web clip, etc.) in `raw/`.
- **Page** — a Markdown file in `wiki/` representing an entity, concept, source summary, comparison, question, or synthesis.
- **Branch** — a domain-specific tag in `frontmatter.branch` declared in `schema.md` (e.g. "architecture", "training").
- **Scaffolding ingest** — special ingest mode for surveys / awesome-lists where many entity/concept stubs are seeded but the source itself isn't filed as a thesis claim.
- **Smart batch** — an ingest mode that finds unprocessed sources in `raw/` (via SHA-256 diff against the cache) and processes them serially with the supervised pause.
- **Cross-vault query** — `/llm-wiki:query --all` or `--vaults a,b`, which loops over registered vaults and synthesizes a multi-section answer.
- **Registry** — `~/.llm-wiki/vaults.json`, the user-level index of known vaults.
- **Research** — the `/llm-wiki:research` workflow: web-search a topic, LLM-triage candidates against `purpose.md`, let the user select sources, ingest them via the standard pipeline.
- **Drift detection** — Step 5.5 of the ingest workflow; checks whether a newly ingested source expands or violates the vault's stated scope and offers a `purpose.md` update prompt.
- **Drift skip** — a source slug recorded in `.llm-wiki/drift-skip.txt` to permanently suppress drift-detection prompts for that source.

## Appendix C — Relevant references

- Karpathy's gist — https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- nashsu/llm_wiki (desktop app, our analysis at `~/src/doc/llm_wiki-architecture.md`)
- lucasastorian/llmwiki — https://github.com/lucasastorian/llmwiki
- qmd — https://github.com/tobi/qmd
- Obsidian — https://obsidian.md
- Pandoc — https://pandoc.org

For decisions log and version history, see [DECISIONS.md](DECISIONS.md).
