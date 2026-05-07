---
description: Safely delete an llm-wiki vault — removes the directory and de-registers it from ~/.llm-wiki/vaults.json. Refuses to remove anything other than a real vault. Always confirms before deleting.
allowed-tools: Bash, Read
argument-hint: <name|path>
---

You are running `/llm-wiki:rm`. This command permanently deletes a vault from disk and removes it from the registry.

**Args:** $ARGUMENTS

---

## Steps

### Step 1 — Parse args

Require exactly one argument: a vault name or a path.

If no argument is provided, print:

```
Usage: /llm-wiki:rm <name|path>

  <name>  — vault name as registered in ~/.llm-wiki/vaults.json
  <path>  — absolute path to the vault directory

Examples:
  /llm-wiki:rm transformer-wiki
  /llm-wiki:rm /Users/me/git/transformer-wiki
```

Then stop.

### Step 2 — Read the registry

Check whether `~/.llm-wiki/vaults.json` exists.

If it does NOT exist, print:

```
Error: No vault registry found at ~/.llm-wiki/vaults.json — nothing to remove.
```

Then stop.

Read `~/.llm-wiki/vaults.json`.

### Step 3 — Resolve name ↔ path

**If the argument looks like a path** (starts with `/`, `~`, or contains `/`):
- Canonicalize it (expand `~` to `$HOME`).
- Search the registry's `vaults` array for an entry whose `path` matches the canonicalized path.
- If found: record both the `name` and `path` from the registry entry.
- If not found: print `"Error: No vault registered at path '<arg>'. Check /llm-wiki:list for registered vaults."` and stop.

**Otherwise treat the argument as a name:**
- Search the registry's `vaults` array for an entry whose `name` matches the argument exactly.
- If found: record both the `name` and `path` from the registry entry.
- If not found: print `"Error: No vault named '<arg>' found in registry. Check /llm-wiki:list for registered vaults."` and stop.

### Step 4 — Hard safety check (no prompt — refuse outright)

Reject immediately if the resolved path matches any of the following. Do NOT ask the user; just refuse and stop:

- `/` (filesystem root)
- `""` (empty string)
- `$HOME` or `/Users/<USER>` — e.g. `/Users/f.luo` (the user's home directory itself)
- `/Users` (the parent of all home dirs)

If any of these match, print:

```
Error: Refusing to delete '<path>' — this path is a system or home directory.
Provide the path to an actual vault subdirectory.
```

Then stop.

### Step 5 — Sanity check: does the path look like a vault?

Run:

```bash
test -f "<resolved-path>/purpose.md"
```

If `purpose.md` is absent, print:

```
⚠ Warning: '<resolved-path>' does not contain purpose.md — this may not be a vault.
Continue anyway? (y/n)
```

Wait for user response. On `n` (or anything other than `y`), print `"Aborted."` and stop.

### Step 6 — Show what will be removed and confirm

Before touching anything, print:

```
Will remove:
  Name:     <name>
  Path:     <resolved-path>
  Registry: entry will be removed from ~/.llm-wiki/vaults.json
  Disk:     <output of: du -sh <resolved-path> 2>/dev/null || echo "(size unknown)">

Confirm deletion? (y/n)
```

Wait for user response. On `n` (or anything other than `y`), print `"Aborted."` and stop.

> **Note on cwd:** If the resolved path is your current working directory, the command will still delete it. After deletion your shell's cwd will be invalid — you will need to `cd` elsewhere manually. The slash command cannot change your shell's working directory.

### Step 7 — Remove the directory

If the path exists on disk:

```bash
rm -rf "<resolved-path>"
```

Print: `"  ✓ Deleted <resolved-path>"`

If the path does NOT exist on disk, print: `"  ℹ Directory <resolved-path> not found on disk — skipping rm (will still clean registry)."`

### Step 8 — Update the registry atomically

1. Read `~/.llm-wiki/vaults.json`.
2. Filter out the entry whose `name` matches the recorded name.
3. Write the updated JSON to `~/.llm-wiki/vaults.json.tmp`.
4. Run `mv ~/.llm-wiki/vaults.json.tmp ~/.llm-wiki/vaults.json`.

Print: `"  ✓ Removed entry '<name>' from ~/.llm-wiki/vaults.json"`

### Step 9 — Print summary

```
Done. Vault '<name>' has been deleted.
  Path removed from disk: <resolved-path>
  Registry entry removed: <name>
```

If any remaining vaults are registered, print: `"Run /llm-wiki:list to see remaining vaults."`

---

## Edge cases

- **Arg matches no name AND no path in registry** — handled in Step 3; tell the user and exit cleanly.
- **Path not on disk but in registry** (user manually deleted the directory) — skip `rm -rf` in Step 7; still remove the registry entry in Step 8; report both.
- **Registry entry exists but path was already manually removed** — same as above.
- **Vault is the current working directory** — warn the user in Step 6 (see note). Proceed normally; the slash command cannot `cd` away. The user must do so after deletion.
- **`~/.llm-wiki/vaults.json` becomes empty after removal** — write `{"vaults": []}` (valid JSON) rather than an empty file.

---

## What this command does NOT do

- Does NOT log to `wiki/log.md` — the vault is being deleted; there is nowhere to log. The output printed above is the audit trail.
- Does NOT push to git or create tags — those are the controller's responsibility.
- Does NOT prompt for which vault if the arg is ambiguous — name matches are exact.
