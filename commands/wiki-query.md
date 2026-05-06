---
description: Answer a question against the current llm-wiki vault, with optional cross-vault mode (--all or --vaults a,b)
allowed-tools: Bash, Read, Write, Edit, Skill, Glob, Grep
argument-hint: "<question>" [--all | --vaults a,b,c] [--vault <path>]
---

You are running `/wiki-query`. Use the `llm-wiki` skill (`@skills/llm-wiki/SKILL.md`) for the full query workflow, vault layout, citation syntax, and qmd integration.

**Args:** $ARGUMENTS

**Parse args:**
- `<question>` (required) — the question to answer; quote if multi-word
- `--all` (mutually exclusive with `--vaults`) — query every vault in `~/.llm-wiki/vaults.json`
- `--vaults a,b,c` (mutually exclusive with `--all`) — query the named subset (must exist in registry)
- `--vault <path>` — override cwd-based vault detection (single-vault mode only)

If neither `--all` nor `--vaults` is provided: single-vault mode (cwd-detected or `--vault`).

---

## Single-vault mode

### Step 0 — Detect the vault.

`--vault <path>` if provided. Else walk up from cwd to find `purpose.md`. Fail with the "Not a vault" error from SKILL.md otherwise.

### Step 1 — Read `<vault>/wiki/index.md` first

This is the LLM's primary navigation anchor. Skim it to identify candidate pages.

### Step 2 — Decide retrieval strategy

Count pages in `<vault>/wiki/` (rough: `find <vault>/wiki -name '*.md' -type f | wc -l`).

- If `qmd` is on PATH AND page count ≥ 100: run `qmd search "<question>" --top 8` (or equivalent). Re-rank results by relevance.
- Else: select pages by index-scan + filename match. Read top 5–10 candidate pages.

### Step 3 — Read the selected pages

Plus always read `<vault>/wiki/synthesis.md` and `<vault>/purpose.md` for cross-cutting context.

### Step 4 — Answer the question

Cite via `[[page-slug]]` (within-vault syntax). Citations should bottom out at `wiki/sources/<slug>.md` pages where possible (so the user can trace back to the original source).

### Step 5 — Offer to file back if non-trivial

If the answer is a substantive comparison, novel synthesis, or new framing (not just a lookup), offer to file back as one of:
- `<vault>/wiki/comparisons/<slug>.md`
- `<vault>/wiki/questions/<slug>.md`
- An entry appended to `<vault>/wiki/synthesis.md`

Suggest the page type based on the answer's shape; ask before filing. If user agrees, write the page and update index.md.

### Step 6 — Append to `<vault>/wiki/log.md`

```
## [<today>] query | <question>
- (one bullet on what was answered or where it landed)
```

---

## Cross-vault mode (`--all` or `--vaults a,b`)

### Step 0 — Read the registry

Read `~/.llm-wiki/vaults.json`. If file is absent or empty, error: "No vaults registered. Run `/wiki-init <name>` first." If a target named in `--vaults a,b` isn't found in the registry, error with the available names.

### Step 1 — Validate vault paths

For each target vault, check the path exists and contains `purpose.md`. Skip with a warning if a vault path is gone.

### Step 2 — Per-vault query

For each target vault, run the per-vault query workflow above (Steps 1–4 of single-vault mode). Collect each vault's contribution: cited pages, key facts, the relevant content of the answer.

### Step 3 — Compose the multi-section answer

Format as:

```
## From <vault-name-1>

<contribution from this vault, with [[wiki-name-1:page-slug]] citations>

## From <vault-name-2>

<contribution from this vault, with [[wiki-name-2:page-slug]] citations>

## Cross-cutting synthesis

<the meta-answer that spans the vaults; surface agreements, disagreements, and complementary framings explicitly>
```

Use `[[wiki-name:page-slug]]` for cross-vault citations (the colon-separated form).

### Step 4 — Filing back is opt-in and explicit

```
This cross-cutting synthesis seems valuable. File to:
(a) <vault-name-1>/wiki/synthesis.md (append)
(b) <vault-name-2>/wiki/synthesis.md (append)
(c) Save without filing
(d) Discard
```

Cross-vault answers must NOT silently land in any single wiki's synthesis.md.

### Step 5 — Append to each queried vault's log.md

For each vault that contributed:
```
## [<today>] cross-query | <question>
- contributed: <one-line summary of what this vault provided>
```

---

**Citation syntax discipline:**

- Within a single vault: `[[page-slug]]` (Obsidian-compatible)
- Across vaults: `[[wiki-name:page-slug]]` (our addition)
- Bare `[[page-slug]]` in a cross-vault answer is ambiguous — always qualify.

**Greeting / non-research questions:** if the user's question is conversational (e.g. "hi") or doesn't seem to need wiki retrieval, skip Steps 1–3 and answer directly. Do NOT log conversational exchanges as `query` entries.
