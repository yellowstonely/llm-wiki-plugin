---
description: Print the registry of known llm-wiki vaults
allowed-tools: Bash, Read
argument-hint: (no arguments)
---

You are running `/llm-wiki:list`. Use the `llm-wiki` skill (`@skills/llm-wiki/SKILL.md`) for context on the registry.

**Steps:**

1. Check whether `~/.llm-wiki/vaults.json` exists.
   - If it does NOT exist, output: `No vaults registered yet. Run /llm-wiki:init <name> [scenario] ["context"] to create one.` Then stop.
2. Read the registry file.
3. For each entry, verify the `path` exists on disk.
4. Render a table to the user with columns: **Name**, **Path**, **Added**, **Status** (`OK` or `MISSING` if path is gone), **Purpose excerpt** (truncated to 80 chars).
5. If any entries are `MISSING`, suggest: `To remove stale entries, edit ~/.llm-wiki/vaults.json by hand or run /llm-wiki:list --prune (TODO: not in v1)`.

**Output format example:**

```
Registered llm-wiki vaults:

| Name                  | Status | Path                                     | Added       | Purpose                                  |
|-----------------------|--------|------------------------------------------|-------------|------------------------------------------|
| transformer-wiki      | OK     | /Users/f.luo/git/transformer-wiki        | 2026-05-06  | Deep dive on transformer architectures…  |
| llm-world-models-wiki | OK     | /Users/f.luo/git/llm-world-models-wiki   | 2026-04-30  | Personal research wiki on world models…  |
```

If no vaults registered or registry file absent, output the helpful message above and stop.
