---
description: Research a topic on the web, triage candidate sources against the vault's purpose, and ingest the chosen ones — for bootstrapping a fresh wiki or expanding coverage of an existing one
allowed-tools: Bash, Read, Write, Edit, Skill, Glob, Grep, WebFetch
argument-hint: "<topic>" [--max-sources N] [--unsupervised] [--vault <path>]
---

You are running `/llm-wiki:research`. Use the `llm-wiki` skill (`@skills/llm-wiki/SKILL.md`) for vault layout, page conventions, ingest workflow, and document-extraction tooling. The skill is the authoritative procedure reference; this slash command is the entry point.

**Args:** $ARGUMENTS

## Argument parsing

- `<topic>` (required) — the research topic; quote multi-word topics (e.g. `"scaling laws for LLMs"`)
- `--max-sources N` (optional, default `20`) — cap on how many web search results to retrieve and triage
- `--unsupervised` (optional) — skip per-source discussion pauses during ingest (Step 4 of single-source workflow); still shows the curate step in Step 3 and always asks before ingesting
- `--vault <path>` (optional) — override cwd-detected vault

---

## Step 0 — Detect the vault

Use `--vault <path>` if provided, else walk up from cwd to find `purpose.md` (stopping at `$HOME` or git root). If not found, refuse with the "Not a vault" error from SKILL.md and suggest `/llm-wiki:init` or `--vault <path>`.

Once the vault root is found, read `<vault>/purpose.md` in full. You will use its headline question and scope fields to triage candidates in Step 2.

**Edge case — unfilled `purpose.md` template:** If `purpose.md` still contains the italicized placeholder prompts (`*<...>*`) from `/llm-wiki:init`, warn the user:
> "Warning: `purpose.md` appears to be an unfilled template. Triage may be weak — relevance scoring will use the section headings as grounding. Consider filling in `purpose.md` before running research for best results."

Proceed anyway, using whatever substantive content is present.

---

## Step 1 — Web search

Run a web search for the topic using firecrawl:

```bash
firecrawl search "<topic>" --limit <max-sources> --json -o /tmp/wiki-research-<slug>.json
```

Where `<slug>` is the topic kebab-cased (e.g. `scaling-laws-for-llms`).

If firecrawl is unavailable (not on PATH or auth fails), fall back to the WebSearch tool with the same query. Adapt result parsing as needed — the goal is: title, url, snippet for each candidate.

Parse the JSON output into a list of candidates, each with:
- `title` — page title
- `url` — canonical URL
- `snippet` — search-result excerpt or description

**Edge case — 0 results:** If the search returns no results, report:
> "No candidates found for '<topic>'. Try broadening the search query or using different keywords."

Do not proceed to Step 2. No log entry is written.

---

## Step 2 — LLM-triage candidates

For each candidate from Step 1, score it against `purpose.md`:

**Relevance score (1–5 stars):** How well does this source match the vault's headline question and scope-in? Use:
- ★★★★★ — directly addresses the headline question; clearly in scope
- ★★★★ — closely relevant; likely useful; minor tangents
- ★★★ — moderately relevant; useful context but not core
- ★★ — loosely related; marginal value
- ★ — off-topic or out of scope

For papers: arxiv URLs and known academic venues (NeurIPS, ICML, ICLR, JMLR, ACL, EMNLP, CVPR, ICCV) default to ★★★★ unless the title or snippet clearly indicates they are off-topic.

**Type tag:** Classify each source as one of:
- `paper` — arxiv links, peer-reviewed conference/journal papers
- `blog` — substack, medium, personal sites, company engineering blogs
- `news` — mainstream press (TechCrunch, VentureBeat, Wired, NYT, etc.)
- `talk` — YouTube talks, recorded conference presentations, podcasts
- `spec` — RFCs, standards documents, technical specifications
- `other` — anything that doesn't fit the above

**Already-covered check:** For each candidate, check whether the URL or a title-derived slug already exists in `<vault>/wiki/sources/`. Use:

```bash
ls <vault>/wiki/sources/
```

If the URL domain + path or the title slug matches an existing source page, mark it `[already-covered: <existing-slug>]`.

---

## Step 3 — Present curated list

Sort candidates by relevance score descending (ties: papers before blogs before others). Display:

```
Found <N> candidate sources for "<topic>". Triaged against your wiki's headline question:

> "<short excerpt of the headline question from purpose.md>"

[1] ★★★★★ (paper, 2022) Hoffmann et al. "Training Compute-Optimal Large Language Models"
    <https://arxiv.org/abs/2203.15556>
    Snippet: We investigate the optimal model size and number of tokens for training a transformer language model...

[2] ★★★★★ (paper, 2020) Kaplan et al. "Scaling Laws for Neural Language Models"
    <https://arxiv.org/abs/2001.08361>
    Snippet: We study empirical scaling laws for language model performance on the cross-entropy loss...

[3] ★★★★ (blog) "Chinchilla revisited" — Inverse Stochastic
    <https://example.com/chinchilla-revisited>
    Snippet: A re-analysis of the Chinchilla paper's conclusions under different compute budgets...

[4] ★★ (news) "Big Tech vs scaling debate" — VentureBeat — [already-covered: big-tech-scaling]
    <https://venturebeat.com/...>
    Snippet: Industry observers weigh in on whether scaling will continue to deliver returns...

...

Which to ingest? Pass any of:
  • "1,3,5"           — specific indexes
  • "1-5"             — a range
  • "all"             — every candidate
  • "all stars 4+"    — all with at least 4 stars
  • "skip blogs"      — exclude by type
  • "none"            — abort without ingesting
```

**Edge case — all already-covered:** If every candidate is marked `[already-covered: ...]`, report:
> "No new sources for '<topic>' — all candidates already exist in the wiki. Try broadening the query, or run `/llm-wiki:query "<topic>"` to see what's already known."

Do not proceed to Step 4. No log entry is written.

---

## Step 4 — User selection (always-ask-first)

**Wait for user input.** Do NOT auto-select or proceed without explicit user response.

Parse the user's selection:
- `"1,3,5"` → indexes 1, 3, 5
- `"1-5"` → indexes 1 through 5 inclusive
- `"all"` → all candidates
- `"all stars 4+"` → all with relevance score ≥ 4 stars
- `"skip blogs"` → all candidates minus those with type `blog`
- `"none"` → abort; do nothing; write no log entry

After parsing, confirm:
> "Proceeding with <N> sources: <comma-separated list of titles>.
> Standard supervised mode — you'll see a discussion pause per source to direct emphasis.
> Use `--unsupervised` to skip pauses if you want to batch through."

If `--unsupervised` was passed, confirm instead:
> "Proceeding with <N> sources in unsupervised mode — no per-source discussion pause. Ingesting now."

Skip any sources already marked `[already-covered: ...]` unless the user explicitly included them by index. If an already-covered source is selected, note:
> "Skipping [<N>] '<title>' — already covered as `<slug>`. Run `/llm-wiki:ingest <url> --re-ingest` to force."

---

## Step 5 — Fetch + ingest each source

For each chosen source (in the order selected):

### 5a. Fetch the source

Fetch via firecrawl scrape to get clean markdown:

```bash
firecrawl scrape "<url>" --format markdown -o <vault>/raw/clips/<slug>.md
```

Slug = URL-derived kebab-case: strip protocol and `www.`, take the last meaningful path segment (or the domain if path is `/`), kebab-case it. Examples:
- `https://arxiv.org/abs/2203.15556` → `arxiv-2203-15556`
- `https://example.com/chinchilla-revisited` → `chinchilla-revisited`

If firecrawl scrape is unavailable, fall back to WebFetch and save the result to `<vault>/raw/clips/<slug>.md`.

**Edge case — fetch fails for one URL:** Log the failure, continue with remaining sources, and summarize all failures at the end:
> "Fetch failed for <N> source(s): <list of titles + URLs>. The rest were ingested successfully."

### 5b. Run the standard ingest workflow

The source is already resolved and in Markdown form at `<vault>/raw/clips/<slug>.md` — Steps 1 (resolve) and 3 (extract) of the SKILL.md ingest workflow are already complete. Enter the ingest workflow at **Step 2 (cache check)**:

1. Compute SHA-256 of the saved file at `<vault>/raw/clips/<slug>.md`. Check `.llm-wiki/ingest-cache.json`. If the hash matches and `--re-ingest` is not set, skip with the "Already ingested" message and continue to the next source — **do not re-ingest**.
2. If cache miss (or `--re-ingest`), continue from **SKILL.md ingest Step 4 (read + discuss)** — NOT Step 1. Read the markdown source. Summarize key takeaways (1–2 paragraphs). Identify proposed entity/concept/comparison/question pages (5–15 typical). Note contradictions with existing wiki content. Describe how this source affects `synthesis.md`.
3. **Discussion pause** — UNLESS `--unsupervised` is set. Pause and wait for user emphasis direction before writing. If `--unsupervised` is set, proceed directly with best-judgment content.
4. Write wiki pages (SKILL.md ingest Step 5):
   - `<vault>/wiki/sources/<slug>.md` (full source page with frontmatter per SKILL.md conventions)
   - Entity / concept / comparison / question pages (create or merge as appropriate)
   - Update `<vault>/wiki/index.md`
   - Update `<vault>/wiki/synthesis.md` if this source materially shifts the cross-cutting position (standard mode only)
   - Append per-source log entry to `<vault>/wiki/log.md`: `## [<today>] ingest | <source-title>`
5. Update SHA-256 cache — SKILL.md ingest Step 6 (atomic write to `.llm-wiki/ingest-cache.json.tmp`, then `mv`).
6. If qmd is present — SKILL.md ingest Step 7: `which qmd && qmd index --update <vault> 2>&1 || true`

---

## Step 6 — Final research log entry

After all sources have been processed, append a single summary entry to `<vault>/wiki/log.md`. This entry is IN ADDITION to the per-source `ingest` entries already written in Step 5.

```
## [<today>] research | <topic>
- Searched web for "<topic>" → <N> candidates found
- User selected <M> sources to ingest: <comma-separated short titles>
- Skipped <K> already-covered, <R> user-rejected
- Wiki touched: see individual ingest entries above
```

Replace all placeholders with actual counts and titles from this run.

---

## Edge cases

**Vault has no `purpose.md` (fresh `/llm-wiki:init`, unfilled template):** Use whatever content is present in `purpose.md` as triage grounding. Warn the user that triage quality will be weak and suggest filling in the template before running research (see Step 0 above).

**Web search returns 0 results:** Report "No candidates found for '<topic>'" and suggest broadening the query. Do not proceed. No log entry is written (see Step 1 above).

**All candidates are `already-covered`:** Report "No new sources for '<topic>' — all candidates already in the wiki" and suggest broadening the query or running `/llm-wiki:query "<topic>"` to see what's known. Do not proceed. No log entry is written (see Step 3 above).

**User selects "none" or aborts at Step 4:** Do nothing. Write no log entry. Leave the vault unchanged.

**Source fetch fails for one URL:** Continue with remaining sources. Do not abort the whole batch. Summarize all failures at the end of Step 5 (see Step 5a above).
