---
description: List all sources ingested into the active llm-wiki vault, with title, year, authors, and branch tag from each source page's frontmatter.
allowed-tools: Bash, Read, Glob
argument-hint: [--vault <path>]
---

You are running `/llm-wiki:sources`. Read-only command — no writes, no log entry. Use the `llm-wiki` skill (`@skills/llm-wiki/SKILL.md`) for vault detection and frontmatter conventions.

**Args:** $ARGUMENTS

## Argument parsing

- `--vault <path>` — explicit vault path, overrides the cwd-walk detection. Optional.

No other flags in v1. If you want filtering, use `grep` on the output, or fall back to `/llm-wiki:query "list sources about X"`.

## Workflow

### Step 1 — Detect the vault

1. If `--vault <path>` was passed, use that. Expand `~` to `$HOME` and resolve to an absolute path. Refuse if the directory does not exist or has no `purpose.md`.
2. Otherwise, walk up from cwd looking for the nearest `purpose.md`, stopping at `$HOME` or a git root. Use that directory.
3. If no vault is found, emit the **Not a vault** error from SKILL.md (`"This directory is not an llm-wiki vault — no purpose.md found..."`) and stop.

### Step 2 — Find source pages

```bash
find <vault>/wiki/sources -maxdepth 1 -type f -name "*.md" | sort
```

If the directory does not exist or contains no `.md` files, print:

```
No sources yet in <vault-name>.

Run /llm-wiki:ingest <url-or-path> to add one, or /llm-wiki:research "<topic>"
to find candidates from the web.
```

Then stop.

### Step 3 — Extract frontmatter from each page

For each source page, parse the YAML frontmatter block (between leading `---` and the next `---`) and extract these fields. All are optional — show `—` if missing:

- `title` (string)
- `year` (string or int)
- `authors` (list of strings; render as `"Vaswani et al."` when ≥ 3 authors, `"Vaswani & Shazeer"` for 2, `"Vaswani"` for 1)
- `branch` (string; the schema-declared branch tag)

The slug is the filename minus `.md`.

If a page's frontmatter is malformed (missing delimiters, invalid YAML), include it in the output with title cells filled as `—` and add a footnote line at the end: `⚠ N source page(s) have malformed frontmatter — run /llm-wiki:lint to surface details.`

### Step 4 — Render the output

Print a Markdown table sorted alphabetically by slug:

```
Sources in <vault-name> (N total):

| Slug | Title | Year | Authors | Branch |
|---|---|---|---|---|
| vaswani-2017 | Attention Is All You Need | 2017 | Vaswani et al. | architecture |
| ...
```

After the table, print a one-line summary:

```
N sources · <count-of-distinct-branches> branches · last ingested: <YYYY-MM-DD>
```

Where `last ingested` is the max `updated:` field across all source pages (or `—` if none have an `updated` field).

### Step 5 — Done

This command is read-only. Do **not**:
- Write or modify any file
- Append to `wiki/log.md`
- Update `~/.llm-wiki/vaults.json`
- Re-index qmd

The output is the entire effect of the command.

## Edge cases

- **Long titles or author lists** truncate gracefully — Markdown tables handle wrapping fine in modern terminals; do not pre-truncate. If a user wants compact output, they can pipe to `awk`/`cut`.
- **No `wiki/sources/` directory at all** (vault hasn't been ingested into yet) — same behavior as empty: print the "No sources yet" message.
- **Sources outside `wiki/sources/`** (e.g. user manually filed a source page elsewhere) — not listed by this command. v1 only enumerates the canonical directory. Document as a known limitation; do not add a recursive search.
- **Malformed YAML in one or more pages** — include those rows with `—` for missing fields, surface a count at the end, suggest `/llm-wiki:lint`.

## What this command does NOT do

- Does NOT show entity / concept / comparison / question pages — only `wiki/sources/`. For a global page index, see `wiki/index.md`.
- Does NOT show raw inputs (`raw/sources/` or `raw/clips/`) — those are the LLM's input layer, not the curated wiki view. To list those: `ls <vault>/raw/sources/` or `ls <vault>/raw/clips/`.
- Does NOT cross-vault aggregate (no `--all` flag in v1). One vault at a time.
