Manage the StreamNext workspace repos — clone missing repos or add a new one.

**Usage:**
- `/setup` — clone all repos in `repos.json` that aren't on disk yet
- `/setup <github-url> [local-folder-name]` — add a new repo to the workspace

**Examples:**
- `/setup`
- `/setup https://github.com/StreamNext/iOS.git`
- `/setup https://github.com/StreamNext/iOS.git ios`

---

The user has run `/setup $ARGUMENTS`.

## No-argument mode (sync missing repos)

If `$ARGUMENTS` is empty:

1. Read `repos.json` and get the `repos` array.
2. For each entry, check whether a directory at `path` exists in the workspace root (the directory containing this `repos.json`).
3. For each missing directory, run `git clone <url> <path>` from the workspace root.
4. Print a summary:
```
StreamNext workspace sync

  [ok]     Backend  — cloned
  [skip]   Proto    — already exists
  ...

Done.
```
Stop here — do not modify `repos.json` or `StreamNext.code-workspace`.

---

## Add-repo mode (URL provided)

If `$ARGUMENTS` is non-empty, parse it:
- First token → `<url>`. Must start with `https://github.com/` and end with `.git`. If not, stop and show the expected format.
- Second token (optional) → `<folder>`. If omitted, derive from the URL's last path segment minus `.git` (e.g. `iOS.git` → `iOS`).

Perform these steps in order — stop and report clearly if any step fails:

### 1. Check for duplicates
Read `repos.json`. If an entry already has the same `url` or `path`, stop and tell the user which field conflicts.

### 2. Update `repos.json`
Append to the `repos` array:
```json
{
  "name": "<folder>",
  "url": "<url>",
  "path": "<folder>",
  "description": ""
}
```
Write back with 2-space indentation.

### 3. Clone (if not already present)
Check whether `<folder>` already exists in the workspace root.
- If not: run `git clone <url> <folder>` from the workspace root.
- If yes: note it was already present, skip cloning.

### 4. Update `.gitignore`
Read `.gitignore`. Append `<folder>/` under the "Repos cloned into workspace" comment block.

### 5. Update `StreamNext.code-workspace`
Read the workspace file. Append to the `folders` array:
```json
{
  "name": "<folder>",
  "path": "<folder>"
}
```
Write back with 2-space indentation.

### 6. Confirm
```
Added <folder> to the StreamNext workspace.

  repos.json                 — new entry added
  .gitignore                 — <folder>/ added
  StreamNext.code-workspace  — new folder added
  <folder>/                  — cloned fresh  (or: already existed)

Reload VS Code to see the new folder (Ctrl+Shift+P → Reload Window).
```
