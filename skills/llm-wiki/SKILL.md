---
name: llm-wiki
description: Use when the user wants to bootstrap, maintain, query, or lint an llm-wiki vault — a personal knowledge base where Claude continuously builds and maintains structured Markdown pages from source documents (PDFs, papers, articles, web clips, notes). Triggers on phrases like "ingest this paper", "ingest this URL", "what does the wiki say about X", "query the wiki", "lint the wiki", "create a new wiki for X", "set up a knowledge base for Y", or whenever the user is in a directory containing purpose.md.
---

# LLM Wiki Skill

## What this skill does

This skill encodes Karpathy's LLM Wiki pattern as a reusable set of workflows for Claude Code. The pattern inverts the default RAG workflow: instead of synthesizing answers from raw chunks at query time, synthesis happens at **ingest time** and is written to disk as structured Markdown. Each subsequent query reads an already-compiled wiki — cross-references are pre-computed, contradictions are pre-flagged, and knowledge compounds with every new source.

### Three-layer architecture

The pattern rests on three layers:

1. **Raw sources (immutable).** PDFs, articles, web clips, transcripts, notes live in `raw/`. The LLM reads from this layer but never modifies it.
2. **Wiki (LLM-maintained Markdown).** Per-source summaries, entity pages, concept pages, comparisons, an evolving synthesis — all in `wiki/`, all cross-linked via Obsidian-style `[[wikilinks]]`. The LLM owns this layer. The human reads and edits.
3. **Schema (governance).** `purpose.md` (the vault's *why* — domain, headline question, scope) and `schema.md` (the vault's *how* — taxonomy, branches, conventions) govern how the wiki grows. This SKILL.md is the third piece of this layer: it encodes the pattern-level workflows that remain constant across every vault.

### Three operations

- **Ingest** — drop a source into `raw/`, the LLM reads it, summarizes, updates 5–15 entity/concept/source pages, appends a log entry. Always supervised: the LLM pauses after summarizing to ask for emphasis direction before writing.
- **Query** — ask a question; the LLM reads `wiki/index.md` first, drills into relevant pages, synthesizes an answer with `[[wikilink]]` citations bottoming out at `wiki/sources/` pages. Non-trivial answers can be filed back as new wiki pages.
- **Lint** — health check: orphan links, dangling pages, index drift, frontmatter validity, schema conformance, synthesis lag, stale claims, missing concept pages. Produces a severity-grouped report with fix options.

Plus bootstrap: `/llm-wiki:init` creates a new vault from scratch, scaffolds the directory tree, writes `purpose.md` and `schema.md` templates or LLM-drafted content from free-form context, runs `git init`, and registers the vault in the user-level registry at `~/.llm-wiki/vaults.json`.

### The operating philosophy

> *"Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase."*

The human works in Obsidian — graph view, plugins, manual edits. The LLM works in Claude Code. They edit the same Markdown files. The maintenance burden (updating cross-references, merging overlapping notes, flagging contradictions) belongs to the LLM, which does not get bored and does not forget to update a cross-reference.

### Slash commands as explicit triggers

All operations have explicit slash commands:

- `/llm-wiki:init <name> [<scenario>] ["<free-form context>"]` — bootstrap a new vault
- `/llm-wiki:ingest <url-or-path> [--scaffold] [--re-ingest] [--all]` — ingest a source
- `/llm-wiki:query <question> [--all | --vaults a,b]` — query the wiki
- `/llm-wiki:research "<topic>" [--max-sources N] [--unsupervised]` — web search + LLM triage + ingest
- `/llm-wiki:lint` — health-check the current vault
- `/llm-wiki:list` — print all registered vaults
- `/llm-wiki:sources [--vault <path>]` — list all sources of the active vault
- `/llm-wiki:rm <name|path>` — safely delete a vault and de-register it

Each command loads this skill. Natural-language phrasings (e.g., "ingest this paper", "what does the wiki say about X") also auto-trigger the skill via the description above.

---

## Vault layout

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
│   └── ingest-cache.json               ← SHA-256 idempotency cache
└── .git/                               ← skill runs `git init` during /llm-wiki:init
```

---

## Vault detection

**Vault detection.** A directory is an llm-wiki vault if it contains `purpose.md` at its root. To detect the active vault, walk up from cwd to the nearest directory containing `purpose.md` (stopping at `$HOME` or the git root). If none is found AND the user invokes any operation other than `/llm-wiki:init`, refuse politely and point at `/llm-wiki:init` or `--vault <path>`.

Any operation that accepts `--vault <path>` uses the specified path directly without walking.

---

## Page conventions

### Filenames

All wiki page filenames are `kebab-case.md`. Derive the slug from the page title: lowercase, spaces to hyphens, strip punctuation. Examples: `transformer-architecture.md`, `scaled-dot-product-attention.md`, `vaswani-2017-attention.md`.

### Cross-references

- Within a vault: `[[page-slug]]` (Obsidian-compatible wikilink)
- Across vaults: `[[wiki-name:page-slug]]` (cross-vault citation syntax; only emitted by cross-vault queries)
- Frontmatter `related:` and `sources:` arrays use slug strings, e.g. `["transformer-architecture", "scaled-dot-product-attention"]`

### Required frontmatter

Every wiki page must include:

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

The `branch:` field is required if the vault's `schema.md` declares branches; omit it otherwise.

### Source-specific additions

For `type: source` pages, add these fields after the required block:

```yaml
authors: [First Last, First Last]
year: YYYY
venue: <publication / website / conference>
url: <canonical URL>
key_claims: ["claim one", "claim two", "claim three"]
```

### Default page-types table

| Type | Purpose | Lives in |
|---|---|---|
| `entity` | A specific named thing — a model, person, lab, dataset, product | `wiki/entities/` |
| `concept` | An abstract idea, technique, theory, phenomenon | `wiki/concepts/` |
| `source` | A summary of one ingested document | `wiki/sources/` |
| `comparison` | Explicit side-by-side analysis of two or more entities/concepts | `wiki/comparisons/` |
| `question` | An open question with the wiki's evolving best answer | `wiki/questions/` |
| `synthesis` | The cross-cutting position file (`synthesis.md`) | `wiki/` (top-level file) |

Scenarios can override or extend this set. The `reading` scenario adds `character`, `theme`, `chapter`, `plot-thread`, `quote`. The `personal` scenario adds `goal`, `journal-entry`, `habit`, `insight`. The `business` scenario adds `customer`, `project`, `decision`, `retrospective`.

---

## Scenarios

| Scenario | Domain examples | Default page types | Default branches |
|---|---|---|---|
| **research** ★ default | "deep dive on transformers", "do LLMs have world models", "competitive analysis of X" | entity, concept, source, comparison, question, synthesis | architecture, training, evaluation, applications, history, cross |
| **reading** | "Frankenstein 1818 vs 1831", "all of Calvino in 2026", book club tracking | character, theme, chapter, plot-thread, quote, source | by-book, themes, chronology |
| **personal** | "self-improvement journal", "fitness + nutrition tracking", life areas | goal, journal-entry, person, habit, insight, source | life-areas (per domain) |
| **business** | "team OKR retrospective", "customer-research backlog", "project decisions" | customer, project, decision, retrospective, person, source | by-project, decisions, people |
| **general** | trip planning, hobby deep-dive, anything open-ended | note, source, person, concept | (none by default) |

All scenarios share `wiki/index.md`, `wiki/log.md`, `wiki/synthesis.md`. Only `purpose.md` and `schema.md` defaults differ per scenario.

---

### research scenario template

When `/llm-wiki:init` runs with scenario `research` and no free-form context, write this content to `purpose.md`:

```markdown
# Purpose

## Domain

*<The topic or area being studied — e.g. "transformer architectures and attention mechanisms", "the role of world models in LLMs">*

## Headline question

*<The one question this wiki is meant to help answer — e.g. "Do large language models develop internal world models, and if so, what form do they take?">*

## Scope — in

*<What's included — specific subtopics, time ranges, source types. Be concrete.>*

## Scope — out

*<What's deliberately excluded. Scope-control discipline: a good "scope out" section prevents wiki sprawl.>*

## Evolving thesis

*<Your current best understanding of the headline question. "No thesis yet" is fine for a fresh wiki. Revise this as the wiki grows — it should change.>*

## Sources strategy

*<Where new sources come from — e.g. arxiv subscriptions, specific authors to follow, podcast feeds, advisor recommendations, periodic literature reviews.>*
```

And write this content to `schema.md`:

```markdown
# Schema

## Branches

*<Define 4–6 thematic branches for this research domain. Each branch becomes a `branch:` frontmatter value that organizes pages by sub-area. Example:>*
- architecture — model design, attention, FFN, normalization
- training — optimization, data, scaling laws
- evaluation — benchmarks, behavior probes
- applications — NLP, vision, audio, multimodal
- history — chronology and lineage
- cross — cross-cutting findings

## Page types

*(Uses default page types from llm-wiki skill: entity, concept, source, comparison, question, synthesis.)*

## Custom rules

*<Any vault-specific conventions beyond the skill defaults — additional frontmatter fields, citation styles, etc.>*
```

---

### reading scenario template

When `/llm-wiki:init` runs with scenario `reading` and no free-form context, write this content to `purpose.md`:

```markdown
# Purpose

## Book(s)

*<The book or books being tracked — title, author, edition/year if relevant. Example: "Mary Shelley's Frankenstein, both the 1818 and 1831 editions".>*

## Themes and threads

*<The themes or reading threads you are tracking across the book(s) — e.g. "the ethics of creation", "Romantic vs Gothic tension", "authorial self-censorship between editions".>*

## Reading position

*<Where you currently are in the reading — chapter, part, percentage. Update this as you read.>*

## Reading goals

*<What you want to get out of this wiki — e.g. "prepare for a book club discussion", "write a comparative essay", "just understand the text better".>*

## Sources strategy

*<Secondary sources, criticism, annotations you plan to bring in — e.g. "SparkNotes for structure, then peer-reviewed literary criticism via JSTOR for depth".>*

## Notes on approach

*<Any reading conventions for this wiki — e.g. "quote pages use the 1818 edition unless marked 1831", "character pages track arc by chapter".>*
```

And write this content to `schema.md`:

```markdown
# Schema

## Branches

*<Organize by book if reading multiple, or by theme. Examples:>*
- by-book — one sub-branch per title if multi-book
- themes — recurring motifs and thematic threads
- chronology — narrative timeline or chapter sequence

## Page types

*(Extends default page types with reading-specific types:)*
- character — a character in the narrative
- theme — a recurring motif or thematic concern
- chapter — per-chapter summary (optional; use for dense or complex texts)
- plot-thread — a narrative thread tracked across chapters
- quote — a significant passage with analysis

## Custom rules

*<Reading-specific conventions — e.g. citation format for quotes, how to handle multiple editions.>*
```

---

### personal scenario template

When `/llm-wiki:init` runs with scenario `personal` and no free-form context, write this content to `purpose.md`:

```markdown
# Purpose

## Life areas

*<The life areas or domains this wiki covers — e.g. "health and fitness", "career development", "relationships and community", "finances". List all areas tracked.>*

## Goals

*<Your current active goals across these areas — e.g. "run a half-marathon by September", "read 24 books in 2026", "build a consistent sleep routine". Goals should be revisable.>*

## Journaling cadence

*<How often and in what format you plan to add journal entries — e.g. "weekly reflection every Sunday", "after significant events only", "daily log, brief".>*

## Scope — in

*<What kinds of inputs belong here — book notes, workout logs, reflection entries, saved articles, conversation notes.>*

## Scope — out

*<What stays out — e.g. "no work projects (those have their own wiki)", "no financial specifics".>*

## Tracking intentions

*<What patterns, habits, or changes you are trying to track over time — e.g. "energy levels vs sleep", "mood vs exercise", "reading retention".>*
```

And write this content to `schema.md`:

```markdown
# Schema

## Branches

*<Organize by life area. One branch per domain you declared in purpose.md. Example:>*
- health — fitness, nutrition, sleep
- learning — books, courses, skills
- relationships — people, community, social
- career — work reflection, skills, aspirations

## Page types

*(Extends default page types with personal-specific types:)*
- goal — a concrete goal with status and milestones
- journal-entry — a dated reflection or log entry
- habit — a recurring practice being tracked
- insight — a realized pattern, lesson, or realization
- person — a significant person in your life (optional)

## Custom rules

*<Any personal conventions — e.g. "journal entries are never edited after the day they're written", "habits include a streak counter in frontmatter".>*
```

---

### business scenario template

When `/llm-wiki:init` runs with scenario `business` and no free-form context, write this content to `purpose.md`:

```markdown
# Purpose

## Team and project focus

*<The team, project, or business domain this wiki covers — e.g. "consumer-graph team Q2 OKRs", "customer research backlog for the offer-intake product", "engineering decision log for the data platform".>*

## Decision-log emphasis

*<What kinds of decisions are worth recording here — e.g. "all architecture decisions with their alternatives considered", "product direction pivots", "process changes affecting the whole team".>*

## Key people and roles

*<Stakeholders, team members, or customer personas relevant to this wiki — helps the LLM contextualize entities correctly.>*

## Scope — in

*<What business artifacts belong here — meeting notes, customer interview summaries, retrospective findings, OKR progress updates, design docs.>*

## Scope — out

*<What stays out — e.g. "no PII", "no financial specifics", "operational runbooks live elsewhere".>*

## Cadence

*<How often this wiki gets updated — e.g. "after every team meeting", "weekly OKR review", "after each customer interview".>*
```

And write this content to `schema.md`:

```markdown
# Schema

## Branches

*<Organize by project, workstream, or business function. Example:>*
- by-project — one sub-branch per active project
- decisions — architectural and product decisions
- customers — customer research and personas
- retrospectives — team retrospectives and learnings
- people — stakeholders and roles

## Page types

*(Extends default page types with business-specific types:)*
- customer — a customer, persona, or account
- project — an active or archived project
- decision — a recorded decision with context, alternatives, and rationale
- retrospective — a team retrospective or post-mortem
- person — a stakeholder or team member

## Custom rules

*<Business-specific conventions — e.g. "decision pages must include an 'alternatives considered' section", "customer pages must not include PII".>*
```

---

### general scenario template

When `/llm-wiki:init` runs with scenario `general` and no free-form context, write this content to `purpose.md`:

```markdown
# Purpose

## What this wiki is for

*<Describe in a sentence or two what you are trying to learn, track, or build here. This is intentionally open-ended — anything goes.>*

## Main question or goal

*<If there is a central question or outcome you are working toward, state it. It's fine to leave this vague and let it sharpen as the wiki grows.>*

## Scope — in

*<What kinds of sources and notes belong here.>*

## Scope — out

*<What doesn't belong — helps prevent sprawl.>*

## Notes on approach

*<Any conventions or preferences for how this wiki should be organized or maintained.>*
```

And write this content to `schema.md`:

```markdown
# Schema

## Branches

*(No default branches for the general scenario. Add branches here if a natural taxonomy emerges after a few ingests.)*

## Page types

*(Uses default page types from llm-wiki skill: entity, concept, source, comparison, question, synthesis. Add scenario-specific types below if needed.)*

## Custom rules

*<Any vault-specific conventions — frontmatter additions, naming rules, citation styles.>*
```

---

## Cross-vault behavior

### Registry location

```
~/.llm-wiki/vaults.json
```

### Registry format

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

### Citation syntax

Within a single vault, use bare wikilinks: `[[page-slug]]`. Across vaults (only in cross-vault query results), use the namespaced form: `[[wiki-name:page-slug]]`. For example, `[[transformer-wiki:scaled-dot-product-attention]]` cites the `scaled-dot-product-attention` page in the `transformer-wiki` vault.

### When the registry is read

Only `/llm-wiki:query --all` and `/llm-wiki:query --vaults a,b` read the registry. Single-vault queries never consult it.

### When the registry is written

Only `/llm-wiki:init` writes to the registry — it appends an entry for the newly created vault (creating `~/.llm-wiki/vaults.json` if it doesn't exist). The user can also hand-edit the file at any time.

### Missing paths

If a registered vault's `path` no longer exists on disk when a cross-vault query runs, skip that vault and emit a warning: `"Warning: vault 'X' at <path> not found on disk — skipping."` Leave the registry entry in place; do not silently delete it.

---

## Workflow procedures

### Log entry types

Every operation appends to `wiki/log.md` with the format `## [YYYY-MM-DD] <type> | <title>`. The `<type>` is one of:

| Type | When emitted |
|---|---|
| `init` | `/llm-wiki:init` creates a new vault |
| `ingest` | `/llm-wiki:ingest` adds a source in standard mode |
| `scaffold` | `/llm-wiki:ingest --scaffold` files a source as a field-map |
| `re-ingest` | `/llm-wiki:ingest --re-ingest` re-processes an already-ingested source |
| `query` | `/llm-wiki:query` (single-vault) is run; entry appended even if no answer is filed back |
| `cross-query` | `/llm-wiki:query --all` or `/llm-wiki:query --vaults a,b` is run; entry appended to each queried vault's log |
| `research` | `/llm-wiki:research` is run; one summary entry per research session, in addition to per-source `ingest` entries |
| `lint` | `/llm-wiki:lint` is run; one entry summarizing the issue count and any auto-fixes |

Greppable via `grep "^## \[" wiki/log.md | tail -10`.

### Ingest workflow

**Mode.** If `--scaffold` is set, run in *field-map mode*: file the source as a field-map (not a thesis source); write entity/concept/question pages as one-paragraph stubs only; use `scaffold` as the log entry type instead of `ingest`; never update `synthesis.md`. Otherwise run in *standard mode*.

**Single-source workflow:**

1. **Resolve the source.**
   - URL: fetch via firecrawl skill if available; fall back to WebFetch (HTML) or `curl` + Read (PDFs). Save extracted Markdown to `raw/clips/<slug>.md`. Slug = filename minus extension, kebab-cased.
   - Path inside the vault: leave in place.
   - Path outside the vault: copy into `raw/sources/`.

2. **Compute SHA-256 of source content. Check `.llm-wiki/ingest-cache.json`.**
   - If the hash matches and `--re-ingest` is not set: print "Already ingested, hash unchanged — pass `--re-ingest` to force." and exit.
   - Otherwise proceed.

3. **Extract to Markdown if needed** (see Document extraction section below). Output staged at `raw/clips/<slug>.md`.

4. **Read the source. Discuss with the user.** Summarize key takeaways in 1–2 paragraphs; ask for emphasis direction. **Pause and wait.** Never barrel through silently. This is Karpathy's supervised pattern — the pause is a feature.

5. **After user response, write wiki pages** (content depth depends on mode):
   - Write `wiki/sources/<slug>.md` (standard: citation + key claims + evidence quality + branch tag; scaffold: field-map summary only).
   - Create or merge `wiki/entities/`, `wiki/concepts/`, `wiki/comparisons/`, `wiki/questions/` pages — typically 5–15 pages touched (standard: rich content; scaffold: one-paragraph stubs only).
   - Update `wiki/index.md` to list the new source page and any new entity/concept pages.
   - Update `wiki/synthesis.md` only if the source materially changes the cross-cutting position (standard mode only — scaffold mode never updates synthesis). Be explicit about what changed and why.
   - Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | <title>` (standard), `## [YYYY-MM-DD] scaffold | <title>` (scaffold), or `## [YYYY-MM-DD] re-ingest | <title>` (when `--re-ingest` is set), with 3–5 bullets describing what was written.

5.5. **Purpose drift detection.** After writing wiki pages but before updating the cache, compare the ingested source against `purpose.md`:
   - Read `purpose.md`. Extract the domain and scope-in sections (or scenario equivalents). Note any scope-out language.
   - Check `<vault>/.llm-wiki/drift-skip.txt` — if the current source slug appears there, skip this step entirely.
   - Examine the source page's `tags` + `key_claims` and any new pages created in Step 5. Judge whether the source materially expands into a topic outside the stated scope or explicitly contradicts the stated scope-out. Lean toward **no prompt** for borderline cases.
   - **If drift detected:** prompt the user with `y/n/s`. On `y`: draft a proposed `purpose.md` update (expanding domain or scope-in), show the full updated file, ask `Save? (y/n/edit)` — never auto-save. On `n`: continue silently. On `s`: append `<source-slug>: skip-drift` to `<vault>/.llm-wiki/drift-skip.txt` (create if absent) and continue.
   - **If no drift:** continue silently.

6. **Update SHA-256 cache.** Write to `<vault>/.llm-wiki/ingest-cache.json` using this format:

   ```json
   {
     "raw/sources/vaswani-2017-attention.pdf": {
       "sha256": "127c8dcb...",
       "last_ingested": "2026-05-06",
       "files_written": [
         "wiki/sources/vaswani-2017-attention.md",
         "wiki/entities/transformer-architecture.md"
       ]
     }
   }
   ```

   - **Cache key** = vault-relative path of the source *after resolution* (i.e. after a URL has been fetched and saved to `raw/clips/<slug>.md`, or an external file copied into `raw/sources/`). URLs themselves are NOT cache keys — the resolved local file is.
   - **Atomic write**: write to `.llm-wiki/ingest-cache.json.tmp`, then `mv` over the original. This protects against corruption on interruption.
   - On `--re-ingest`, the entry is replaced (not merged) — old `files_written` list is overwritten.

7. **If qmd is present:** run the qmd lifecycle block — lazy-register the vault as a qmd collection if missing, then `qmd update` and `qmd embed` (both scoped to the vault's collection). The skill manages the qmd collection automatically; the user never runs qmd commands by hand. See "qmd integration" below for the exact commands.

**Batch workflow (via `--all` or natural-language "ingest all new sources"):**

1. Walk `raw/sources/` and `raw/clips/`, list every file.
2. Cross-reference `.llm-wiki/ingest-cache.json` — find files whose hash is not cached or differs from the cached value.
3. Print: `"Found N unprocessed sources: <list>. Ingest all in order?"`
4. **Always ask first.** Wait for user confirmation. The user can say "yes", "skip B", "do A and D only", etc.
5. For each confirmed source, run the standard supervised single-source workflow above. The discussion pause stays in place per source. The user can move quickly ("emphasis: same, proceed") or pause to direct.

---

### Query workflow

**Single-vault workflow:**

1. Read `wiki/index.md` first.
2. If qmd is present and the wiki has 100 or more pages: run `qmd query "<question>" -c <vault-name> -n 8`, where `<vault-name>` is `basename <vault>`. `qmd query` is hybrid (BM25 + vector) plus rerank — best quality available.
3. Otherwise: select relevant pages by index scan and filename match (typically 5–10 pages).
4. Read those pages. Also read `wiki/synthesis.md` and `purpose.md` for cross-cutting context.
5. Answer with `[[wiki-page]]` citations that bottom out at `wiki/sources/` pages.
6. If the answer is non-trivial (a comparison, novel synthesis, new framing): offer to file as `wiki/comparisons/<slug>.md`, `wiki/questions/<slug>.md`, or as an entry in `wiki/synthesis.md`. Suggest the page type based on shape; ask before filing. When filing as an append to `synthesis.md`, bump the `updated:` field in `synthesis.md`'s frontmatter to today's date. (New comparison/question pages carry fresh frontmatter; only the `synthesis.md` append needs this explicit bump.)
7. Append `## [YYYY-MM-DD] query | <question>` to `wiki/log.md` with one bullet on the answer and where it landed (if filed back).

**Cross-vault workflow (via `--all` or `--vaults a,b`):**

1. Read `~/.llm-wiki/vaults.json`. Validate that target vaults exist on disk; warn and skip any that do not.
2. For each target vault: read its `purpose.md`, `wiki/synthesis.md`, and `wiki/index.md`, then run the per-vault query logic (same as above; use qmd if available per that vault).
3. Compose a multi-section answer:
   - **Per vault:** what each wiki contributes, with `[[wiki-name:page-slug]]` citations.
   - **Cross-cutting synthesis:** the meta-answer spanning the vaults, with disagreements and agreements between wikis surfaced explicitly.
4. Filing back is opt-in and explicit. Ask: "This cross-cutting synthesis seems valuable. File to: (a) wiki-A's synthesis, (b) wiki-B's synthesis, (c) save without filing, (d) discard." Cross-vault answers do not silently land in any single wiki's `synthesis.md`.
5. Append `## [YYYY-MM-DD] cross-query | <question>` to each queried vault's `wiki/log.md` with one bullet per vault on what was contributed.

---

### Research workflow

Triggered by `/llm-wiki:research "<topic>"`. Reuses ingest infrastructure for the per-source heavy lifting; the unique parts are web search and triage.

**Mode:** by default, every selected source goes through the standard ingest workflow including the discussion-pause per source. Pass `--unsupervised` on `/llm-wiki:research` to skip per-source pauses (the curated-list confirmation in Step 4 is then the only supervision checkpoint).

**Steps:**

1. **Detect the vault.** Walk up from cwd to find `purpose.md`, or use `--vault <path>`. Refuse with the "Not a vault" error if not found. Read `purpose.md` for relevance grounding.

1.5. **Check `purpose.md` filled-state; offer to draft if unfilled.** Count occurrences of the regex `\*<[^>]+>\*` in `purpose.md`. If 3 or more placeholders are found, the file is still a template stub: prompt the user with three choices — (y) draft `purpose.md` from the research topic now using the same LLM-drafting logic as `/llm-wiki:init`'s free-form-context branch, (n) proceed with weak triage, or (e) edit `purpose.md` manually and re-run. On (y): draft, show the result, ask for confirmation, save on confirm, then continue. On (n): warn that triage may be weak, then continue. On (e): print the vault path and exit cleanly without searching. If fewer than 3 placeholders are found, `purpose.md` is considered filled and this step is a no-op.

2. **Web search.** Use the firecrawl skill (preferred default per the user's global instructions). Limit to `--max-sources N` (default 20). If firecrawl unavailable, fall back to WebSearch. Parse out `{title, url, snippet}` per candidate.

3. **LLM triage.** For each candidate, score against `purpose.md`:
   - **Relevance** (1–5 stars): how well does this match the headline question and scope-in?
   - **Type**: paper | blog | news | talk | spec | other.
   - **Already-covered**: cross-reference URL/title slug against `wiki/sources/*.md`.
   - For arxiv URLs and known academic venues, default to ★★★★ unless clearly off-topic.

4. **Present + ask.** Show curated list sorted by relevance. Always ask before proceeding (Karpathy-faithful supervised pattern). Selection syntax: `1,3,5` / `1-5` / `all` / `all stars 4+` / `skip blogs` / `none`.

5. **Per chosen source:** fetch (firecrawl scrape → markdown), save to `<vault>/raw/clips/<slug>.md`, run the standard ingest workflow (Steps 1–7 of Ingest workflow above). Skip already-covered (cache hit) unless user opted to re-ingest.

6. **Summary log entry.** In addition to the per-source `ingest` entries, append one summary:

   ```
   ## [<today>] research | <topic>
   - Searched web for "<topic>" → N candidates
   - User selected M sources to ingest: <short-titles>
   - Skipped K already-covered, R user-rejected
   ```

**Edge cases:**
- Vault has unfilled `purpose.md`: prompt user to either draft it now from the research topic, proceed with weak triage, or edit it manually first. See Step 1.5 above.
- Search returns 0 results: report and suggest broadening the query.
- All candidates already-covered: report; suggest `/llm-wiki:query "<topic>"` to see what's already there.
- Source fetch fails for one URL: continue, summarize failures at the end.
- User selects "none" or aborts: do nothing, no log entry.

---

### Lint workflow

Run the following 8 checks against the current vault:

| Check | What it detects |
|---|---|
| Orphan `[[wikilinks]]` | Links pointing to non-existent pages — list the missing slugs |
| Dangling pages | Pages with no inbound `[[wikilinks]]` AND not listed in `index.md` (exempts system files: `index.md`, `log.md`, `synthesis.md`) |
| Index drift | Pages in `wiki/` not listed in `index.md`, and vice-versa |
| Stale claims | Source pages with claims contradicted by newer source pages (heuristic; LLM-judged on a small sample) |
| Frontmatter validity | Missing required fields (`type`, `title`, `created`, `updated`, `sources`) |
| Schema conformance | `branch:` values not in `schema.md`'s declared branch list |
| Synthesis lag | `synthesis.md`'s `updated:` field older than the newest source page |
| Missing concept pages | Concepts mentioned in 3 or more pages with no dedicated page anywhere in `wiki/` (any subdirectory) (heuristic) |
| Purpose drift | Sources in `wiki/sources/` whose `tags` + `key_claims` significantly diverge from `purpose.md`'s `## Domain` / `## Scope — in`. If more than ~30% of source pages fall outside the stated scope, flag for review. Sources that explicitly match the stated scope-out are clear violations. (Heuristic — severity: Suggestion.) |

**Report format.** Produce a Markdown report grouped by severity:
- **Error** — broken structure that will confuse navigation (orphan links, frontmatter invalidity)
- **Warning** — degraded quality (index drift, schema non-conformance, synthesis lag)
- **Suggestion** — improvement opportunities (dangling pages, missing concept pages, stale claims)

**User prompt.** After showing the report: "Found N issues. Want me to: (a) fix all auto-fixable, (b) walk through each and decide, (c) just save the report and we'll do it later?"

**Auto-fixable** (no judgment required):
- Remove orphan `[[wikilink]]` markup (replace with plain text)
- Stub missing required frontmatter fields (add `tags: []`, `related: []`, `sources: []` where absent)
- Update `index.md` (add missing entries; remove entries for deleted pages)

**Not auto-fixable** (requires user judgment):
- Stale-claim resolution (LLM can flag and propose, but user confirms)
- Missing-concept-page authoring (new page content; ask before writing)
- Purpose drift — requires deciding whether to update `purpose.md` to match actual content, or relocate off-scope sources to a different vault

After auto-fixes, if qmd is present, run the qmd lifecycle block (same as ingest Step 7) — lazy-register if missing, then `qmd update` + `qmd embed` scoped to the vault.

**Log.** Append `## [YYYY-MM-DD] lint | N issues` to `wiki/log.md` with the count summary broken down by severity.

---

## qmd integration

The skill manages qmd's collection lifecycle automatically. Users never run `qmd collection add`, `qmd update`, or `qmd embed` by hand — the skill does it as a side effect of the operations they already run (`/llm-wiki:ingest`, `/llm-wiki:lint`, `/llm-wiki:rm`).

### Detection

Check `command -v qmd` (or `which qmd`). If absent, proceed silently — never error or warn. qmd is purely optional.

### Collection naming

The collection name is the basename of the vault path. Example: `<vault> = ~/git/transformer-wiki` → collection name is `transformer-wiki`.

### Lifecycle block (used by ingest, lint, research)

After any successful operation that touched `wiki/` files, run this block:

```bash
if command -v qmd >/dev/null 2>&1; then
  vault_name=$(basename <vault>)
  # Lazy registration — first time we touch a vault that has qmd installed
  if ! qmd collection list 2>/dev/null | grep -q "^${vault_name} ("; then
    qmd collection add <vault> --name "${vault_name}" >/dev/null 2>&1 || true
  fi
  qmd update -c "${vault_name}" >/dev/null 2>&1 || true
  qmd embed -c "${vault_name}" >/dev/null 2>&1 || true
fi
```

Every command in this block is best-effort with `|| true`. Never fail the parent operation on a qmd error.

### When to invoke `qmd query` (read path)

| Wiki size | qmd installed | Query strategy |
|---|---|---|
| Any | No | Read `index.md`, then drill into matched pages |
| < 100 pages | Yes | Same as above (qmd is overkill at this scale; index-scan is faster end-to-end) |
| ≥ 100 pages | Yes | `qmd query "<question>" -c <vault-name> -n 8` (hybrid + rerank), then read top hits |

The 100-page threshold matches Karpathy's gist recommendation. The `-c <vault-name>` flag scopes results to the active vault, even if the user has other qmd collections registered.

### Removal hook

`/llm-wiki:rm` runs `qmd collection remove <vault-name>` after deleting the directory and updating the registry. Best-effort; silent on "no such collection".

### Self-healing on first query after late install

If `qmd query` returns "collection not found" (e.g. the user installed qmd after the vault was already substantial), fall back to index-scan for the current query, then run the lifecycle block before the next query. No need to surface this transition to the user — it just starts working.

### v1: CLI only via Bash

Call qmd via the Bash tool. qmd's own MCP server and Claude Code plugin are not used by this skill (per design decision #11). Users who install the qmd plugin separately get qmd-specific commands; they coexist with this skill cleanly.

### Install note

The skill does not install qmd. The `/llm-wiki:init` post-install summary includes: *"Optional: install qmd (https://github.com/tobi/qmd) for fast hybrid search once your wiki passes ~100 pages — the skill manages it for you once installed."*

---

## Document extraction

When `/llm-wiki:ingest` receives a non-Markdown source, extract it to Markdown before reading. Use this priority table:

| Format | First try | Fallback | External dep |
|---|---|---|---|
| `.md`, `.txt` | Read directly | — | none |
| `.pdf` | **`marker_single`** if installed (best quality — equations as LaTeX, tables as Markdown, multi-column layouts), else Claude Code's Read tool | `pdftotext -layout <file> <out>` | marker-pdf (optional, recommended); poppler (last-resort fallback only) |
| `.docx`, `.odt`, `.rtf`, `.epub` | `pandoc <file> -t markdown` | — | **pandoc** |
| `.pptx` | `pandoc -t markdown` | unzip + grep slide XMLs | pandoc |
| `.xlsx`, `.csv` | `python3 -c "import openpyxl; ..."` one-liner | `xlsx2csv` | `python3` |
| `.html` (URL) | firecrawl skill if available, then WebFetch | `curl` + `pandoc` | — |
| `.html` (file on disk) | `pandoc` | — | pandoc |

**Rules:**

- Always invoke Python as `python3`, not `python` — the bare `python` command is unreliable across platforms.
- For URLs: prefer the firecrawl skill (global default for web); fall back to WebFetch for HTML pages; use `curl` + Read for PDFs behind URLs.
- DOCX, PPTX, RTF, and EPUB all require pandoc. If pandoc is absent, emit a clear error and suggest: "Install pandoc (`brew install pandoc`) or pre-convert this file to Markdown and drop the `.md` into `raw/sources/`."
- Output of every extraction lands in `raw/clips/<slug>.md`. Originals stay where they are. The converted Markdown is what the wiki workflow reads.

### PDF extraction priority chain (v0.4.3)

For `.pdf` inputs, try this chain in order. Each tier guards on the next being unavailable; falling all the way through to pdftotext is rare but recoverable:

1. **`marker_single`** (preferred when installed). High-quality ML-based extraction — preserves equations as LaTeX (`$$...$$`), tables as Markdown, multi-column layouts, headings. Invocation:
   ```bash
   tmp=$(mktemp -d)
   marker_single "<src>" --output_dir "${tmp}" --output_format markdown --disable_image_extraction
   # output lives at ${tmp}/<basename>/<basename>.md
   mv "${tmp}/$(basename '<src>' .pdf)/$(basename '<src>' .pdf).md" "<vault>/raw/clips/<slug>.md"
   rm -rf "${tmp}"
   ```
   Detect with `command -v marker_single`. If marker runs but produces an empty/missing file, treat as failure and fall to tier 2.
2. **Claude Code's Read tool** (default fallback). The LLM reads the PDF and writes extracted markdown directly to `<vault>/raw/clips/<slug>.md`. This is the v0.4.2-and-earlier behavior; preserves the skill's working state for users who haven't installed marker.
3. **`pdftotext -layout`** (last resort). Use only if the Read tool fails on a particular PDF (e.g. heavily encrypted/non-standard). Invocation: `pdftotext -layout "<src>" "<vault>/raw/clips/<slug>.md"`.

Speed note: marker takes ~10–30s per page on M-series CPU. For batch ingest of long PDFs, consider whether marker's quality is worth the wait — the user can pre-convert manually with marker outside the skill if they need finer control.

Corporate-proxy note: marker (and other Python tools that download model weights from datalab.to) may need `REQUESTS_CA_BUNDLE` and `SSL_CERT_FILE` set to a CA bundle that includes the corporate-proxy CA. Document in the user's shell profile, not in the skill.

---

## Error handling

**Not a vault.** When any operation other than `/llm-wiki:init` is invoked outside a vault (no `purpose.md` found up to `$HOME` or git root): "This directory is not an llm-wiki vault — no `purpose.md` found. Either `cd` into a vault directory, pass `--vault <path>`, or run `/llm-wiki:init <name>` to create a new one."

**Already exists.** When `/llm-wiki:init` targets a directory that already exists: "Directory `<name>` already exists. `/llm-wiki:init` will not overwrite an existing directory. Choose a different name, or delete the directory first if you want to start fresh."

**Missing extraction tool.** When ingest needs pandoc, python3, or poppler and they are absent: "Cannot extract `<file>` — requires `<tool>` which is not installed. Install it (`brew install <tool>`) or pre-convert the file to Markdown and drop the `.md` into `raw/sources/`."

**Cache hit.** When the source SHA-256 matches the cache and `--re-ingest` is not set: "Source `<file>` was already ingested (hash unchanged). Pass `--re-ingest` to force re-processing."

**Unknown scenario.** When `/llm-wiki:init` receives a scenario name that isn't one of `research | reading | personal | business | general`, list the valid scenarios and ask the user to retry. Do NOT silently default — silent defaults mask typos.

Example response: `"<scenario>" isn't a known scenario. Valid choices: research, reading, personal, business, general. Re-run with one of these.`
