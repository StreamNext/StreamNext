# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is the **workspace root** for StreamNext. It contains no application code. Its purpose is onboarding, tooling, and specs.

The actual code lives in repos cloned inside this directory:

| Folder | Repo | Purpose |
|---|---|---|
| `Backend/` | StreamNext/Backend | C#/.NET REST + JWT API server |
| `Proto/` | StreamNext/Proto | API contracts: OpenAPI (REST transport) + protobuf (shared client message types, if/when needed) |
| `iOS/` | StreamNext/iOS | SwiftUI native iOS app *(repo not created yet)* |
| `Android/` | StreamNext/Android | Kotlin/Compose native Android app *(repo not created yet)* |

`repos.json` is the authoritative list of all repos. `StreamNext.code-workspace` opens everything in one VS Code window.

## Getting started (new contributor)

```bash
# clone everything already in repos.json
/setup
```
(or, without Claude Code: `git clone` each URL in `repos.json` into this directory by hand)

Then open `StreamNext.code-workspace` in VS Code.

## Adding a repo to the workspace

```
/setup <github-url>
/setup <github-url> <local-folder-name>
```

This updates `repos.json`, clones the repo, and adds it to `StreamNext.code-workspace`.

## Claude sessions

Always start from this directory. The sub-repo `CLAUDE.md` files should be imported below via `@` once those repos exist and have one, the same way the four spec files below are — so a session here has full context for every repo without switching directories.

The four spec files below are loaded automatically via `@` imports and are meant to become the authoritative context for architecture, data model, and requirements — **they currently start as skeletons**. Filling them in as real product/architecture decisions get made is part of ongoing work here, not a one-time setup task. For feature work, the `/implement` skill re-reads them explicitly as Step 0.

## Feature development workflow

Every feature request follows these phases in order. Never skip or combine phases without explicit user confirmation.

**Spec step** (before any code)
Read `docs/features/<feature>.md`. If it doesn't exist, use `/spec <feature-name>` to scaffold it, then fill in Overview, Business rules, Clients, and API endpoints with the user before proceeding. Confirm the spec is correct and complete before moving to Phase 1.

**Phase 1 — Understand**
Read all relevant code. Identify every area impacted: contract (OpenAPI/proto), backend layers, client layers, existing tests, adjacent features. Ask clarifying questions before proceeding.

**Phase 2 — Locate & confirm**
State exactly which files will change and why. Wait for explicit user confirmation before writing any code.

**Phase 3 — Implement**
Apply changes across all identified files.

**Phase 4 — Test**
Write unit tests targeting ≥80% coverage on changed code. Tests must validate correctness of behavior, not just that the code compiles.

**Phase 5 — Document**
Update `docs/features/<feature>.md` — set Status to `implemented`, add or update Implementation notes, and append a dated Change log entry. Verify no regressions in adjacent behavior.

See `.claude/commands/implement.md` for the full expanded workflow (multi-repo macro-phase sequencing, per-repo agent delegation, testing checklists), and `.claude/commands/resolve-defect.md` for the equivalent TDD-based workflow for bug fixes.

**Version bump (undefined — decide on the first feature that ships)**

Every repo touched by a feature should get its version bumped and its release notes authored in the same change, same spirit as any mature multi-repo project. The *specifics* (which file holds the version per repo, patch/minor/major convention, release-notes format, author-vs-publish split) aren't decided yet for this project — don't invent them per-feature. Decide once, on the first feature that actually needs it, and record the decision in this section so every future feature just follows it.

## Git workflow (this repo only)

Confirm this repo's own convention (commit directly to `main`, no PR, is a reasonable default for a specs-only repo with nothing to CI-gate) and record it here once confirmed.

```bash
git add <specific files>
git commit -m "docs/chore: meaningful description"
git push origin main
```

Sub-repos (`Backend/`, `iOS/`, `Android/`) will each have their own workflow once they exist — **always read the sub-repo's `CLAUDE.md` before committing or pushing there** once it has one.

**No AI attribution in commits, in this repo or any sub-repo.** Never add a `Co-Authored-By: Claude` (or any AI attribution) trailer to a commit message — just the summary/body, no attribution footer. This is a standing rule across the whole workspace.

## Response format

At the end of every completed response, always include:

1. A short prose summary of what changed and why (1–3 sentences)
2. A tree listing every file that was modified, created, or deleted

## Project context

<!-- Uncomment each line once that spec file has real content worth auto-loading every session:
@docs/specs/00-overview.md
@docs/specs/01-requirements.md
@docs/specs/02-architecture.md
@docs/specs/03-data-model.md
-->

## Feature logic

<!-- @docs/features/<feature>.md entries go here as features are built -->

## Sub-repo context

<!-- @Backend/CLAUDE.md
@Proto/CLAUDE.md
@iOS/CLAUDE.md
@Android/CLAUDE.md
Uncomment each line once that repo exists and has a CLAUDE.md to import. -->
