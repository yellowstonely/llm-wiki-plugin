---
description: Bootstrap a new llm-wiki vault with scenario templates and optional free-form context
allowed-tools: Bash, Read, Write, Edit, Skill, WebFetch
argument-hint: <name> [<scenario>] ["<free-form context>"]
---

You are running `/llm-wiki:init`. Use the `llm-wiki` skill (`@skills/llm-wiki/SKILL.md`) for layout, page conventions, and scenario templates.

**Args:** $ARGUMENTS

## Argument parsing

- `<name>` (required) — directory name. Create as `<cwd>/<name>` (relative) if the name does not start with `/` or `~`. If it starts with `/` or `~`, treat as an absolute path and create it there.
- `<scenario>` (optional, default `research`) — one of: `research | reading | personal | business | general`
- `"<free-form context>"` (optional, must be quoted) — describes the wiki's domain/purpose; if present, draft richer `purpose.md` and `schema.md` via LLM call

Parsing rules:
- If only one positional arg is provided, it is `<name>`; scenario defaults to `research`.
- If two positional args are provided, the second is `<scenario>` (validate against the 5 known scenarios).
- If a quoted string is provided as a third arg, treat it as free-form context.

---

## Workflow

### Step 1 — Validate inputs

1. If `<scenario>` is not one of `research | reading | personal | business | general`, emit the **Unknown scenario** error from SKILL.md, list the valid options, and stop. Do NOT silently default.
2. Resolve the vault path:
   - Relative name → `<cwd>/<name>`
   - Absolute path (starts with `/` or `~`) → use as-is (expand `~` to `$HOME`)
3. If the resolved directory already exists, emit the **Already exists** error from SKILL.md and stop.

### Step 2 — Create directory tree

```bash
mkdir -p <vault>/raw/sources
mkdir -p <vault>/raw/clips
mkdir -p <vault>/raw/assets
mkdir -p <vault>/wiki/entities
mkdir -p <vault>/wiki/concepts
mkdir -p <vault>/wiki/sources
mkdir -p <vault>/wiki/comparisons
mkdir -p <vault>/wiki/questions
```

### Step 3 — Generate `purpose.md`

**If free-form context was provided:**

Draft a rich `purpose.md` via LLM call (you, Claude Code). Infer the domain, headline question, scope-in, scope-out, and initial thesis from the free-form context. Use the chosen scenario's section structure (per the scenario templates in SKILL.md). Do not use placeholder italics — write real content that reflects what the user told you. The draft should be substantive enough that the user could edit it rather than rewrite it from scratch.

Write the LLM-drafted content to `<vault>/purpose.md`.

**If no free-form context:**

Write the scenario's templated stub from SKILL.md to `<vault>/purpose.md`. The templates use `*<your X here>*` prompts in italics so the user knows exactly what to fill in. Copy the template verbatim from SKILL.md — do not invent a new template. Choose the template that matches the chosen scenario:

- `research` → research scenario template
- `reading` → reading scenario template
- `personal` → personal scenario template
- `business` → business scenario template
- `general` → general scenario template

### Step 4 — Generate `schema.md`

**If free-form context was provided:**

Draft a `schema.md` via LLM call. Infer 4–6 starter branches from the domain you identified in Step 3. Write each branch as `- branch-name — short description`. Include the note `(Uses default page types from llm-wiki skill.)` under `## Page types`. Leave `## Custom rules` with an empty placeholder.

Write the LLM-drafted content to `<vault>/schema.md`.

**If no free-form context:**

Write a minimal `schema.md` with these exact sections:

```markdown
# Schema

## Branches

*(none yet — add branches here once a natural taxonomy emerges after a few ingests)*

## Page types

*(Uses default page types from llm-wiki skill: entity, concept, source, comparison, question, synthesis.)*

## Custom rules

*(none)*
```

### Step 5 — Generate `wiki/index.md`

Write the following skeleton to `<vault>/wiki/index.md`:

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

### Step 6 — Generate `wiki/log.md`

Write the following to `<vault>/wiki/log.md`:

```markdown
# <name> log

## [<today>] init | <name> created

- Scenario applied: <scenario>
- Free-form context used: <yes/no>
- Directory tree created: raw/{sources,clips,assets} and wiki/{entities,concepts,sources,comparisons,questions}
- purpose.md and schema.md: <"drafted from free-form context" or "templated stub">
```

Replace `<today>` with today's date in `YYYY-MM-DD` format, `<name>` with the vault name, `<scenario>` with the chosen scenario, and fill the last two bullets accurately.

### Step 7 — Generate `wiki/synthesis.md`

Write the following stub to `<vault>/wiki/synthesis.md`:

```markdown
---
type: synthesis
title: Synthesis
created: <today>
updated: <today>
tags: []
related: []
sources: []
---

# Synthesis

*No synthesis yet. This file will evolve as sources are ingested and cross-cutting patterns emerge. The skill updates it automatically when an ingest materially changes the cross-cutting position.*
```

Replace `<today>` with today's date in `YYYY-MM-DD` format.

### Step 8 — Run `git init` and write `.gitignore`

```bash
git init <vault>
```

Write the following to `<vault>/.gitignore`:

```
.obsidian/workspace*
.obsidian/cache
.DS_Store
*.swp
.llm-wiki/*.tmp
```

### Step 9 — Update the vault registry

1. Ensure `~/.llm-wiki/` directory exists: `mkdir -p ~/.llm-wiki`
2. If `~/.llm-wiki/vaults.json` does not exist, create it with content `{"vaults": []}`.
3. Read the existing registry.
4. Build a new entry:
   ```json
   {
     "name": "<name>",
     "path": "<absolute path to vault>",
     "added": "<today>",
     "purpose_excerpt": "<first ~150 chars of purpose.md body, single-line, no newlines>"
   }
   ```
   For `purpose_excerpt`, strip the `# Purpose` heading and any leading blank lines; take the first ~150 characters of the remaining content; collapse newlines to spaces.
5. **Atomic write:** write the updated JSON to `~/.llm-wiki/vaults.json.tmp`, then `mv ~/.llm-wiki/vaults.json.tmp ~/.llm-wiki/vaults.json`. This protects against corruption on interruption.

### Step 10 — Print summary

Print the following, with all placeholders filled in:

```
Vault created: <absolute path>

Scenario:    <scenario>
Free-form context: <yes / no>

Suggested next steps:
  • Open in Obsidian:          open -a 'Obsidian' <absolute path>
  • Configure Obsidian Web Clipper to write to: <absolute path>/raw/clips/
  • Set Obsidian's Settings → Files & links → Default location for new attachments to: raw/assets
  • Optional: install qmd for fast hybrid search at scale (https://github.com/tobi/qmd)
  • Ingest your first source: /llm-wiki:ingest <url-or-path>
```
