Scaffold or list feature specs in docs/features/.

**Usage:**
- `/spec <feature-name>` — create a new spec file ready to fill in
- `/spec` — list all features and their current state

---

The user has run `/spec $ARGUMENTS`.

## No-argument mode — list features

If `$ARGUMENTS` is empty:

Read every `.md` file in `docs/features/` (excluding `_template.md`). For each file determine its state:
Read the `## Status` line value from each file.

Print a table:
```
Feature specs

  auth.md          implemented
  ...
```

Stop here.

## Create mode — scaffold a new spec

If `$ARGUMENTS` is non-empty, treat it as the feature name.

1. Derive the filename: lowercase, spaces replaced with hyphens + `.md` (e.g. `spending summary` → `spending-summary.md`)
2. Check if `docs/features/<filename>` already exists — if so, tell the user and stop
3. Write the file to `docs/features/<filename>` with this content (replace `<Feature Name>` with the title-cased name and `YYYY-MM-DD` with today's date):

```markdown
# Feature: <Feature Name>

## Status
planned

## GitHub issue
<!-- Optional: GitHub issue tracking this feature, e.g. #42 or https://github.com/org/repo/issues/42 -->

## Overview
<!-- What this feature does and why it exists -->

## Business rules
<!-- Key invariants and constraints the implementation must uphold -->
-

## Clients
<!-- Which clients this feature touches: iOS (iOS), Android (Android), or both.
     A UI-only feature may need just one; a new/changed API endpoint needs Backend and whichever
     client(s) expose it. Don't assume both clients are in scope by default — say which. -->
-

## API endpoints
<!-- REST endpoints involved — not RPCs; the backend is plain REST, contract hand-authored in
     Proto/openapi/v1/. Format: `METHOD /path` — description.
     For a client-only change with no backend change write: "iOS-only, no API changes"
     (or "Android-only", or "iOS + Android", as appropriate). -->
-

## Implementation notes
<!-- Filled in after implementation (Phase 5) -->

## Change log
- YYYY-MM-DD: Initial spec (@mauro)
```

4. Confirm:
```
Created docs/features/<filename>

Fill in Overview, Business rules, and API endpoints — then Claude will proceed with the 5-phase workflow.
```
