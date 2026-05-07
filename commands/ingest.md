---
description: Ingest a source (URL or path) into the current llm-wiki vault — extracts to markdown, runs supervised analysis, writes wiki pages, updates index/log, manages SHA-256 cache. Supports --scaffold for surveys, --re-ingest to force, --all for smart batch.
allowed-tools: Bash, Read, Write, Edit, Skill, Glob, Grep, WebFetch
argument-hint: <url-or-path> [--scaffold] [--re-ingest] [--all] [--vault <path>]
---

You are running `/llm-wiki:ingest`. Use the `llm-wiki` skill (`@skills/llm-wiki/SKILL.md`) for the full ingest workflow, vault layout, page conventions, and document-extraction tooling. The skill is the authoritative procedure reference; this slash command is the entry point.

**Args:** $ARGUMENTS

**Parse args:**
- First positional (or only): `<url-or-path>` — required UNLESS `--all` is passed
- `--scaffold` — field-map mode; generate stubs only (see Mode below)
- `--re-ingest` — force re-processing even when SHA-256 hash is unchanged
- `--all` — smart batch: find all unprocessed sources, ask user, run serially
- `--vault <path>` — override cwd-detected vault

**Mode:** if `--scaffold` is set, this is *field-map mode* — file the source as a survey/awesome-list, generate stub pages only, never update `synthesis.md`. Else *standard mode* — write rich content, update synthesis if warranted.

**Step 0 — Detect the vault.** Use `--vault <path>` if provided, else walk up from cwd to find `purpose.md` (stopping at `$HOME` or git root). If not found, refuse with the "Not a vault" error from SKILL.md and suggest `/llm-wiki:init` or `--vault <path>`.

---

## If `--all` is set or user phrases this as "ingest all new sources"

This is the smart batch entry path. It finds sources that have never been ingested (or whose content has changed since last ingest) and offers to process them.

1. Walk `<vault>/raw/sources/` and `<vault>/raw/clips/`. List every file (recursively, but typically one level deep). Exclude any non-document files (e.g. `.DS_Store`, `*.gitkeep`).
2. Read `<vault>/.llm-wiki/ingest-cache.json` if it exists. If it doesn't, treat every file as unprocessed.
3. For each file, compute SHA-256 of its content. Compare with the cached hash for that vault-relative path. Build the *unprocessed* list = files whose hash is not in cache OR differs from the cached hash.
4. Print: `"Found <N> unprocessed sources: <list with paths and brief content excerpt>. Ingest all in order?"`
5. **Always ask first.** Wait for user confirmation before processing anything. User may say "yes", "skip A", "do A and D only", re-order, etc. Respect all such adjustments.
6. For each confirmed source, run the **single-source workflow below**, in order, with the supervised pause per source. The user can move quickly ("proceed, no special emphasis") or engage deeply per source — the pace is theirs to set.

Note: `--scaffold` and `--re-ingest` flags compose with `--all`. Pass them through to each single-source run.

---

## Single-source workflow

### Step 1 — Resolve the source

Determine what the source is and ensure it lives inside the vault before hashing.

- **URL** → fetch using firecrawl skill if available (the user's global default for web content). If firecrawl is unavailable, fall back to WebFetch for HTML pages or `curl` + Read for PDFs. Save extracted markdown to `<vault>/raw/clips/<slug>.md`, where `<slug>` is:
  - PDF URLs: `kebab-case(filename minus extension)` from the URL path
  - HTML pages: slugified version of the page `<title>` tag
- **Path inside vault** — leave in place. No copy needed.
- **Path outside vault** — copy into `<vault>/raw/sources/`. Refuse if it would overwrite an existing file; suggest a renamed copy or `--re-ingest` if the user intends to replace.

### Step 2 — Compute SHA-256 and check cache

The cache key is the *post-resolution* vault-relative path (e.g. `raw/clips/vaswani-2017.md` after a URL fetch, or `raw/sources/my-paper.pdf` for a path-based ingest). The URL itself is NOT the cache key — the resolved local file is.

Steps:
- Compute SHA-256 of the resolved file's content using `shasum -a 256 <file>` or equivalent.
- If `<vault>/.llm-wiki/ingest-cache.json` doesn't exist, treat as a cache miss and proceed.
- Read the cache. Look up the vault-relative path.
- If the hash matches the cached value AND `--re-ingest` is not set: print "Already ingested (hash unchanged). Pass --re-ingest to force." and exit cleanly. Do not write anything.
- Otherwise proceed.

### Step 3 — Extract source to markdown if needed

Use SKILL.md's extraction reference. Format-by-format decision table:

| Format | Tool |
|---|---|
| `.md`, `.txt` | Read directly (no extraction step needed) |
| `.pdf` | Read tool first; if it returns empty or garbage, fall back to `pdftotext <file> -` |
| `.docx`, `.odt`, `.rtf`, `.epub` | `pandoc <file> -t markdown -o <vault>/raw/clips/<slug>.md` |
| `.pptx` | `pandoc -t markdown` first; fallback: `unzip -p <file> 'ppt/slides/slide*.xml' \| grep -oE '<a:t>[^<]+</a:t>'` |
| `.xlsx`, `.csv` | `python3 -c "import openpyxl; wb=openpyxl.load_workbook(...)"` style one-liner |
| `.html` (already on disk) | `pandoc -t markdown` |

If a required tool (pandoc / pdftotext / python3) is not installed, emit the "Missing extraction tool" error from SKILL.md and exit. Do NOT silently produce empty content — an empty extraction is worse than no extraction.

### Step 4 — Read the source. Discuss with user.

Read the resolved markdown content in full. Then:

1. Summarize key takeaways in 1–2 paragraphs.
2. Identify proposed entity/concept/comparison/question pages (5–15 typical).
3. Note any contradictions with content already in the wiki.
4. Describe where this source would slot into `synthesis.md` — if at all.

Then **PAUSE AND WAIT** for user input on emphasis direction. **Do NOT silently proceed to writing.** This pause is a feature, not a formality — it is Karpathy's "supervised" pattern. The user might redirect: "focus on the training regime, skip the background", "add a comparison with GPT-2 while you're at it", "skip synthesis update this time". Honor all such direction.

### Step 5 — After user response, write wiki pages

The exact content depth depends on mode: `--scaffold` = stubs only; standard = rich content.

**5a. Source page** at `<vault>/wiki/sources/<slug>.md`:

Frontmatter fields required:
```yaml
type: source
title: "<paper or article title>"
created: <today YYYY-MM-DD>
updated: <today YYYY-MM-DD>
tags: [...]
related: [list of page slugs]
sources: ["<original-source-filename>"]
authors: [First Last, ...]
year: YYYY
venue: <publication / website / conference>
url: <canonical URL if any>
key_claims: ["claim one", "claim two", "claim three"]
```

Body:
- Standard mode: rich summary, key claims, evidence quality, what's contested or novel, any caveats.
- Scaffold mode: field-map only — what topics/entities does this source survey, at what depth, for what audience. No thesis derivation.

**5b. Entity / concept / comparison / question pages** at `<vault>/wiki/<type>/<slug>.md` (5–15 pages typical):

- For pages that **already exist**: MERGE by (a) appending the new source slug to the `sources:` frontmatter array (deduplicate), (b) adding new content sections without overwriting prior content, (c) bumping `updated:` to today.
- For **new pages**: write fresh content with full frontmatter. Standard mode = substantive paragraphs. Scaffold mode = one-paragraph stubs only.

**5c. Update `<vault>/wiki/index.md`:** add the new source page and any new entity/concept/comparison/question pages to their respective sections. Keep alphabetical order within each section.

**5d. Update `<vault>/wiki/synthesis.md`** — STANDARD MODE ONLY. Update only if this source materially changes the cross-cutting position. Be explicit about what changed and why. Bump `updated:` to today. If nothing changed, leave `synthesis.md` untouched. Scaffold mode: never touch `synthesis.md`.

**5e. Append to `<vault>/wiki/log.md`:**
```
## [<today>] <type> | <source-title>
- <what pages were written/created>
- <what pages were merged/updated>
- <any contradictions or review flags>
```
Where `<type>` is:
- `ingest` — standard mode, first-time ingest
- `scaffold` — `--scaffold` mode
- `re-ingest` — `--re-ingest` flag was set (regardless of mode)

### Step 6 — Update SHA-256 cache

Atomic write to `<vault>/.llm-wiki/ingest-cache.json`:

1. Read existing cache JSON, or start from `{}` if absent.
2. Set (or overwrite) the entry for the vault-relative path:
   ```json
   {
     "raw/sources/vaswani-2017-attention.pdf": {
       "sha256": "127c8dcb...",
       "last_ingested": "2026-05-06",
       "files_written": [
         "wiki/sources/vaswani-2017-attention.md",
         "wiki/entities/transformer-architecture.md",
         "wiki/concepts/multi-head-attention.md"
       ]
     }
   }
   ```
3. Write to `.llm-wiki/ingest-cache.json.tmp`, then `mv` over the original. This protects against corruption on interruption (atomic rename).

On `--re-ingest`, the entry is replaced, not merged. Old `files_written` list is overwritten with the new one.

### Step 7 — qmd reindex (if available)

```bash
which qmd && qmd index --update <vault> 2>&1 || true
```

Don't fail the ingest if qmd is missing or errors. The `2>&1 || true` suppresses non-zero exits.

---

## Edge cases

**Re-ingest replaces, not merges.** When `--re-ingest` is set and the source was previously ingested, re-run the full workflow. On completion, replace the cache entry entirely — `files_written`, `sha256`, and `last_ingested` are all overwritten. Do not attempt to diff old vs new wiki output; write the best current pages and let git track the diff.

**Malformed frontmatter on merge.** During Step 5 entity/concept-page merging, if an existing page's frontmatter is malformed (missing delimiters, invalid YAML, wrong type for a field), surface the issue explicitly rather than silently overwriting. Tell the user: "Page `<path>` has malformed frontmatter — describe the issue. Offer to fix it before merging." Do not proceed with a silent overwrite.

**User abort during Step 4.** If the user says "skip" or "abort" after the Step 4 summary, roll back partial state:
- Delete any file fetched or extracted during this invocation from `raw/clips/` (only files created in this run, never pre-existing ones).
- Do NOT update `ingest-cache.json`.
- Do NOT write or modify anything in `wiki/`.
Leave the vault in the exact state it was in before Step 1.
