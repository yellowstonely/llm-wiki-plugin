# LLM Wiki

Karpathy's LLM Wiki pattern as a Claude Code plugin.

---

## What it does

The default LLM-with-documents workflow is RAG: upload sources, ask a question,
retrieve chunks at query time, synthesize an answer. It works fine for one-off
questions — but knowledge never compounds. The 50th query is no faster, no
smarter, and no more aware of cross-document connections than the 5th. Every
time you ask "how does X compare to Y across these five papers?" the LLM has to
find and stitch together fragments from scratch.

LLM Wiki inverts the pattern. **Synthesis happens at ingest time.** When you
drop a source into the vault, Claude reads it, writes entity pages, concept
pages, a source summary, and updates an evolving `synthesis.md`. The next query
reads the already-compiled wiki — cross-references are pre-computed,
contradictions are pre-flagged, and the wiki gets richer with every source.

The framing is Karpathy's: *"Obsidian is the IDE; the LLM is the programmer;
the wiki is the codebase."* You open Obsidian (graph view, plugins, manual
edits). Claude opens a Claude Code session. Both edit the same plain-Markdown
files. The maintenance burden disappears because Claude doesn't get bored,
doesn't forget to update a cross-reference, and can touch 15 files in one pass.

---

## Install

```bash
git clone https://github.com/f-luo/llm-wiki-plugin ~/.claude/plugins/llm-wiki
```

Reload Claude Code; the plugin auto-discovers and loads. No other setup
required for the core workflow.

**Optional dependencies:**

- `pandoc` — required for DOCX, PPTX, RTF, EPUB ingest
  (`brew install pandoc`)
- `python3` — required for XLSX/CSV ingestion (standard on macOS)
- `qmd` — hybrid semantic search for large wikis (≥ ~100 pages);
  see [https://github.com/tobi/qmd](https://github.com/tobi/qmd)

PDFs, Markdown, plain text, and HTML work without either dependency.

---

## Quickstart

```
/wiki-init transformer-wiki research "deep dive on transformer architectures, attention mechanisms, and scaling laws - headline question: what made the transformer design win?"
/wiki-ingest https://arxiv.org/pdf/1706.03762
/wiki-query "how does attention compare to gated recurrence?"
/wiki-research "scaling laws"  # find more sources via web search
```

After `/wiki-init`, open the vault directory in Obsidian. Run `/wiki-ingest`
for each source. Use `/wiki-query` to ask questions; non-trivial answers can be
filed back into the wiki automatically.

---

## Commands

### `/wiki-init`

**Syntax:** `/wiki-init <name> [<scenario>] ["<free-form context>"]`

Bootstrap a new wiki vault. Creates the full directory tree, scaffolds
`purpose.md` and `schema.md` (either from a scenario template or drafted by
Claude from your free-form description), initializes `index.md`, `log.md`,
`synthesis.md`, runs `git init`, and registers the vault in
`~/.llm-wiki/vaults.json`.

- `<name>` — directory name; created in the current working directory (or as an
  absolute path). Refuses if the directory already exists.
- `<scenario>` — one of `research | reading | personal | business | general`.
  Defaults to `research`.
- `"<free-form context>"` — optional prose description of the wiki's domain,
  headline question, and scope. If provided, Claude drafts a richer
  `purpose.md` and seeds branches in `schema.md` from your description; if
  omitted, both files get scenario-templated stubs with section headers and
  prompts to fill in.

**Example:**

```
/wiki-init fitness-wiki personal "tracking strength training and nutrition - goal: identify what actually moves the needle on progressive overload"
```

---

### `/wiki-ingest`

**Syntax:** `/wiki-ingest <url-or-path> [--scaffold] [--re-ingest] [--all] [--vault <path>]`

Ingest a source into the current vault. Resolves the source (URL fetch,
copy from outside the vault, or use as-is if already inside), computes a
SHA-256 hash to skip unchanged sources, extracts to Markdown if needed, then
summarizes and discusses with you before writing wiki pages. The supervised
pause is intentional — Karpathy's pattern: one source at a time, directed
emphasis.

After you respond, Claude writes a source summary page, creates or updates
entity and concept pages (typically 5–15 files), updates `index.md`, and
appends a log entry. `synthesis.md` is updated only when the source materially
shifts the cross-cutting position.

- `--scaffold` — field-map mode: seed many entity/concept stubs (one paragraph
  each) without treating the source as a thesis claim. Useful for surveys and
  awesome-lists.
- `--re-ingest` — force re-processing even if the SHA-256 hash matches the
  cache.
- `--all` — smart batch: scan `raw/sources/` and `raw/clips/` for files not
  yet in the cache, present the list, and ingest them in order (per-source
  pause stays in place).

**Example:**

```
/wiki-ingest raw/sources/vaswani-2017-attention.pdf
/wiki-ingest https://lilianweng.github.io/posts/2023-01-27-the-transformer-family-v2/
/wiki-ingest --all
```

---

### `/wiki-query`

**Syntax:** `/wiki-query <question> [--all | --vaults a,b] [--vault <path>]`

Answer a question against the wiki. Reads `index.md` first, selects relevant
pages by index scan (or `qmd search` if qmd is installed and the wiki has
≥ ~100 pages), reads those pages plus `synthesis.md` and `purpose.md`, then
answers with `[[wikilink]]` citations that bottom out at `sources/` pages.
If the answer is a novel comparison or synthesis, Claude offers to file it
back as a new page — and asks before writing.

- `--all` — query every vault in `~/.llm-wiki/vaults.json`. Produces a
  multi-section answer: per-vault contributions plus a cross-cutting synthesis.
  Citations use `[[wiki-name:page-slug]]` form.
- `--vaults a,b` — query a named subset of registered vaults.

**Example:**

```
/wiki-query "what are the main critiques of scaled dot-product attention?"
/wiki-query --vaults transformer-wiki,llm-world-models-wiki "do transformers learn world models?"
```

---

### `/wiki-research`

**Syntax:** `/wiki-research "<topic>" [--max-sources N] [--unsupervised] [--vault <path>]`

Searches the web for the topic, triages candidates against the vault's `purpose.md`, presents a curated list with star ratings, and ingests the chosen ones. Useful for bootstrapping a fresh wiki ("/wiki-init transformer-wiki research" then "/wiki-research 'scaling laws in transformer training'") or expanding coverage of an existing one. Per-source supervision pauses are on by default; pass `--unsupervised` to batch through.

**Example:**

```
/wiki-research "scaling laws in transformer training"
```

Returns a triaged list of papers/blogs/news/etc. with relevance ratings; you pick which to ingest.

---

### `/wiki-lint`

**Syntax:** `/wiki-lint [--vault <path>]`

Health-check the current vault. Walks every page in `wiki/`, checks for:
orphan `[[wikilinks]]` pointing to missing pages, dangling pages with no
inbound links, index drift (pages not listed in `index.md`), frontmatter
validity (missing required fields), schema conformance (branch tags not
declared in `schema.md`), synthesis lag (`synthesis.md` older than the newest
source page), and missing concept pages (concepts mentioned in 3+ pages
without their own page).

Produces a Markdown report grouped by severity (error / warning / suggestion)
and offers three choices: auto-fix all fixable issues, walk through each one,
or save the report and defer. Auto-fixable: orphan-link cleanup, frontmatter
field stubbing, index updates. Not auto-fixable: stale-claim resolution,
missing concept pages (require judgment).

**Example:**

```
/wiki-lint
```

Lint operates on the cwd-detected vault only (or `--vault <path>`). Cross-vault
lint is not in v1.

---

### `/wiki-list`

**Syntax:** `/wiki-list`

Print the contents of `~/.llm-wiki/vaults.json` in a readable format. Shows
each registered vault name, path, date added, and purpose excerpt. Marks
vaults whose paths no longer exist on disk. Useful when you can't remember
vault names before running `/wiki-query --vaults ...`.

**Example:**

```
/wiki-list
```

---

## Scenarios

Five scenario presets ship with the plugin. The scenario you choose at
`/wiki-init` determines the default page types and the structure of the
`purpose.md` and `schema.md` templates. All scenarios share `index.md`,
`log.md`, and `synthesis.md`.

| Scenario | Typical domain | Default page types |
|---|---|---|
| **research** ★ default | Deep dives, paper reviews, competitive analysis | entity, concept, source, comparison, question, synthesis |
| **reading** | Book clubs, literary analysis, reading programs | character, theme, chapter, plot-thread, quote, source |
| **personal** | Journaling, fitness, self-improvement tracking | goal, journal-entry, person, habit, insight, source |
| **business** | OKR retrospectives, customer research, decision logs | customer, project, decision, retrospective, person, source |
| **general** | Trip planning, hobby deep-dives, open-ended learning | note, source, person, concept |

---

## Vault layout

`/wiki-init` creates this tree:

```
<vault>/
├── purpose.md              ← the wiki's why (domain, headline question, scope, thesis)
├── schema.md               ← taxonomy: branches, page-type overrides, custom rules
├── raw/
│   ├── sources/            ← user-curated PDFs, DOCX, notes (immutable)
│   ├── clips/              ← Obsidian Web Clipper output + ingest-time extractions
│   └── assets/             ← inline images (Obsidian attachment folder target)
├── wiki/
│   ├── index.md            ← content catalog (skill maintains)
│   ├── log.md              ← append-only chronological log (skill maintains)
│   ├── synthesis.md        ← evolving cross-cutting position (skill maintains)
│   ├── entities/           ← people, models, organizations, datasets
│   ├── concepts/           ← theories, methods, abstractions
│   ├── sources/            ← per-source summary pages
│   ├── comparisons/        ← side-by-side analyses
│   └── questions/          ← open questions with evolving answers
├── .obsidian/              ← created by Obsidian on first open (not by skill)
├── .llm-wiki/
│   └── ingest-cache.json   ← SHA-256 idempotency cache (skill sidecar, rebuildable)
└── .git/                   ← initialized by /wiki-init
```

The vault is detected by the presence of `purpose.md` in the current directory
(or any parent up to the git root). The `--vault <path>` flag overrides this.

---

## How it relates to Karpathy's gist

This plugin is a direct implementation of
[Karpathy's LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
as a portable Claude Code plugin. The gist describes a 76-line abstract
pattern: raw sources (immutable) → wiki pages (LLM-maintained Markdown) →
schema file (governs behavior). We implement all three layers plus the three
operations (ingest, query, lint) from the gist verbatim.

Our refinements on top of the gist: splitting the single `CLAUDE.md` config
into `purpose.md` (the *why*) and `schema.md` (the *how*), shipping five
scenario presets to eliminate the "what page types should I have?" decision,
adding a SHA-256 idempotency cache and smart-batch ingest to reduce friction,
supporting cross-vault queries via a vault registry with `[[wiki-name:page-slug]]`
citation syntax, and treating `synthesis.md` as a single top-level evolving
position file. We explicitly keep Obsidian as the reading IDE (rather than
building our own UI) and use qmd for search at scale when it's available — both
as Karpathy recommends.

---

## Optional integrations

**Obsidian Web Clipper** — configure the clipper to write to `raw/clips/` in
your vault. The next `/wiki-ingest --all` will pick up the saved clips as
unprocessed sources.

**Obsidian attachment folder** — point Obsidian's "Default attachment folder"
setting at `raw/assets/` so inline images land inside the vault tree.

**qmd hybrid search** — install [qmd](https://github.com/tobi/qmd) for fast
hybrid (BM25 + vector) search. The plugin uses it automatically once your wiki
reaches ~100 pages; below that threshold, index-scan navigation handles queries
well. After each ingest, the plugin runs `qmd index --update <vault>` to keep
the index warm. qmd's MCP server integration is not wired in v1 (the plugin
calls it via the shell); MCP wiring is an opt-in future step.

---

## Limitations

v1 is intentionally narrow. There is no background daemon or file watcher —
the skill runs inside a Claude Code session and requires explicit invocation
for each operation. `/wiki-lint` operates on one vault at a time; cross-vault
contradiction detection is deferred to a future version. Vault discovery is
explicit (registry-based), not automatic filesystem scanning. There is no async
review queue — the supervised pause per source is the intended workflow. Vector
embedding is not built into the skill; qmd handles this when needed and is
entirely optional. The plugin does not include a web clipper of its own (use
Obsidian Web Clipper). Two concurrent Claude Code sessions ingesting into the
same vault simultaneously will race on shared files — don't do that.

---

## License

MIT
