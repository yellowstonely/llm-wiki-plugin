---
description: Health-check the current llm-wiki vault for orphan links, drift, frontmatter issues, and stale claims
allowed-tools: Bash, Read, Edit, Write, Skill, Grep, Glob
argument-hint: [--vault <path>]
---

You are running `/wiki-lint`. Use the `llm-wiki` skill (`@skills/llm-wiki/SKILL.md`) for vault layout,
page conventions, and the lint workflow.

**Args:** $ARGUMENTS (optional `--vault <path>`)

---

## Steps

### 1. Detect the vault

- If `--vault <path>` is provided, use that path as the vault root. Verify it contains `purpose.md`; if
  not, abort with the "Not a vault" error defined in SKILL.md.
- Otherwise, walk up from the current working directory looking for `purpose.md`. If not found at any
  ancestor, abort with the same error.

### 2. Run all checks

Execute the full lint workflow documented in SKILL.md. Checks, in order:

| # | Check | What to look for |
|---|-------|-----------------|
| 1 | **Orphan `[[wikilinks]]`** | Links pointing to slugs that have no corresponding page file |
| 2 | **Dangling pages** | Pages with no inbound `[[wikilinks]]` and not listed in `index.md` (exempts system files: `index.md`, `log.md`, `synthesis.md`) |
| 3 | **Index drift** | Pages present on disk under `wiki/` but absent from `index.md`, or entries in `index.md` with no matching file |
| 4 | **Frontmatter validity** | Pages missing any of the required fields: `type`, `title`, `created`, `updated`, `sources` |
| 5 | **Schema conformance** | `branch:` values that do not appear in `schema.md`'s declared branch list |
| 6 | **Synthesis lag** | `synthesis.md`'s `updated:` field is older than the `updated:` of the newest source page |
| 7 | **Stale claims** | Heuristic LLM judgment: sample source pages with overlapping topics and flag claims that appear contradictory or outdated relative to each other |
| 8 | **Missing concept pages** | Concepts mentioned (by name or `[[slug]]`) in 3 or more pages with no dedicated page anywhere in `wiki/` (any subdirectory) |

### 3. Produce a markdown report

Group findings by severity:

- **Error** — structural problems that break navigation or render the vault inconsistent (orphan links,
  missing required frontmatter, index drift)
- **Warning** — quality issues that degrade usefulness but don't break structure (dangling pages,
  synthesis lag, schema violations)
- **Suggestion** — low-confidence or heuristic findings (stale claims, missing concept pages)

Each finding must include:
- Affected file path (relative to vault root)
- Description of the issue
- Suggested fix where applicable

### 4. Ask the user how to proceed

After presenting the report, prompt:

```
Found <N> issues (<E> errors, <W> warnings, <S> suggestions).

How to proceed?
(a) Auto-fix everything auto-fixable (orphan-link cleanup, frontmatter stubbing, index updates)
(b) Walk through each issue and decide
(c) Save the report to .llm-wiki/lint-<date>.md and exit
```

Wait for the user to choose (a), (b), or (c) before continuing.

### 5. Execute the user's choice

- **(a) Auto-fix all:** Apply every auto-fixable change in one pass. Print a summary of what was changed.
  For items that require manual judgment, leave them unresolved and print a note listing them.
- **(b) Interactive walk-through:** Step through each finding one at a time. For each, show the issue and
  proposed fix (if auto-fixable), then wait for the user to approve, skip, or provide a custom resolution.
- **(c) Save and exit:** Write the report to `.llm-wiki/lint-<date>.md` (ISO date, e.g.
  `lint-2026-05-06.md`) and exit without making any changes.

### 6. Append to `wiki/log.md`

Add a log entry in this format:

```markdown
## [YYYY-MM-DD] lint | N issues

- <summary bullet, e.g. "fixed 5 orphan links; 3 stale-claim items pending review">
```

### 7. Re-index if qmd is available

If `qmd` is on PATH **and** any pages were deleted or renamed during auto-fix (step 5a or 5b), run:

```bash
qmd index --update <vault>
```

---

## Auto-fixable scope

The following issues can be resolved without user judgment:

- **Orphan `[[wikilinks]]`** — strip the `[[]]` link syntax, preserving the alias text if present
  (e.g. `[[missing-slug|Concept Name]]` → `Concept Name`)
- **Missing frontmatter arrays** — add empty `tags: []`, `related: []`, and/or `sources: []` to any page
  that is missing those fields
- **Index drift** — add entries to `index.md` for pages on disk that are absent, and remove entries for
  pages that no longer exist

## NOT auto-fixable (always require user judgment)

- **Stale claims** — requires domain knowledge to determine which claim is correct
- **Missing concept pages** — requires deciding whether the concept warrants its own page
- **Schema-conformance violations** — a non-declared `branch:` value may be an intentional new branch the
  user wants to add to `schema.md`; auto-removal could silently destroy valid metadata
