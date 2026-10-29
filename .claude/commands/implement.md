Begin implementation of a feature spec following the 5-phase workflow.

**Usage:**
- `/implement <feature-name>` — read spec from `docs/features/<feature-name>.md`
- `/implement #<N>` or `/implement <N>` — fetch spec from GitHub issue N in `StreamNext/StreamNext`

**Examples:** `/implement end-to-end-encryption` · `/implement #5` · `/implement 12`

---

The user has run `/implement $ARGUMENTS`.

If `$ARGUMENTS` is empty, stop and show the usage above.

## Step 0 — Read the project specs

Before doing anything else, read all four spec files to ensure full context. Do not skip this step, even if you believe you already know the project state — the specs are the authoritative source of truth for architecture, data model, and requirements.

- `docs/specs/00-overview.md` — what the product is and what repos exist
- `docs/specs/01-requirements.md` — what is implemented vs planned
- `docs/specs/02-architecture.md` — tech stack, layer order, key constraints
- `docs/specs/03-data-model.md` — all entities, columns, ID types, enums, migration history

Read all four in parallel before proceeding. If any of these are still skeletons (freshly scaffolded, not yet describing a real system), say so and treat Step 5 (Understand) as needing to establish more ground truth than usual before proceeding.

---

## Step 1 — Locate the spec

**Determine input type** from `$ARGUMENTS`:

- If `$ARGUMENTS` is a bare number (`3`, `42`) or a hash-prefixed number (`#3`, `#42`), treat it as a GitHub issue reference. Strip any leading `#` to get the issue number.
- Otherwise treat it as a feature name and follow the local-file path below.

### GitHub issue path

Run:
```bash
gh issue view <number> --repo StreamNext/StreamNext --json title,body,labels,type
```

Use the issue **title** as the feature name and the issue **body** as the spec content. If this org has configured GitHub Issue Types (check with `gh api orgs/StreamNext/issue-types` the first time this matters — don't assume either way), and `type` is `Bug`, this is very likely `/resolve-defect`'s job instead; say so and confirm with the user before proceeding as a feature build. There is no local file to read or validate — proceed directly to Step 2 using the issue content.

After fetching, record the issue reference as `#<number>` so Phase 5 can reference it.

Mark the issue as in-progress:
```bash
gh issue edit <number> --repo StreamNext/StreamNext --add-label "in-progress"
```
If this fails because the label doesn't exist, create it first: `gh label create "in-progress" --repo StreamNext/StreamNext --color "fbca04" --description "Work is underway"` — then retry. If it still fails, log the error and continue.

If `gh issue view` fails (issue not found, auth error, etc.), stop and report the error to the user.

### Local file path

Derive the filename: lowercase, spaces to hyphens, `.md` extension.
Check if `docs/features/<filename>` exists. If not, tell the user and suggest running `/spec <feature-name>` first.

Read the spec file. Reject and stop if any of the following are true:
- Overview is empty or still contains only the `<!-- -->` comment placeholder
- Business rules contains only the placeholder comment or a single `-` with no text
- API endpoints contains only the placeholder comment or a single `-` with no text
- Status is already `implemented`

If rejected, tell the user exactly which sections need to be filled in before proceeding.

Update the `## Status` line in the spec file to `in-progress`.

## Step 2 — Validate the spec content

For **both paths**, confirm the spec has enough substance to proceed:
- There is a clear description of what the feature does
- There is at least one business rule or constraint
- There is at least one API endpoint or UI interaction described

If any of these are missing, summarise what's unclear and ask the user to fill in the gaps before continuing.

## Step 3 — Set status to in-progress

**Local file path only:** Update the `## Status` line in the spec file to `in-progress`.

**GitHub issue path:** No local file to update yet — status tracking happens via the issue itself until Phase 5 creates the local spec.

## Step 4 — Determine macro-phase scope

The backend is plain REST + JWT. Read the **API endpoints** section. If any endpoint is new or changed, this feature requires **Macro Phase A (Contract)**: add/update it in `Proto/openapi/v1/` first, hand-authored and PR-reviewed there, before `Backend` implements it. `Proto`'s `.proto` files (if/when this project uses them) are shared *message-type* definitions only for iOS/Android codegen — no `service`/`rpc` blocks are live; touch them only if a client actually needs a new generated type, separate from the REST contract change itself.

State clearly which macro phases this feature touches:
- If the endpoint contract changes: "This feature requires a contract change. Starting **Macro Phase A — Contract**. Backend/iOS/Android work is blocked until the OpenAPI change is reviewed and merged."
- If no contract change: "No contract changes needed. Running the 5 phases against [backend / iOS / Android — name whichever repos are actually affected]."

Determine which client repos this feature actually touches (a bug fix or UI-only feature may only need one):
- **Macro Phase B — Backend** (`Backend/`) — needed for any new/changed endpoint, business rule, or DB change
- **Macro Phase C — iOS** (`iOS/`) — needed for any user-facing change to the native iOS client
- **Macro Phase D — Android** (`Android/`) — needed for any user-facing change to the native Android client

iOS/Android are independent of each other (neither blocks the other) but both are blocked on Backend when the feature depends on a new/changed endpoint. Don't assume a feature needs both clients — check the spec's actual UI surface before scoping; most features land on one platform first and the other catches up separately, not necessarily in the same `/implement` run.

## Step 5 — Phase 1: Understand

Read the spec in full. Then read all relevant code across the impacted repos:
- The OpenAPI contract in `Proto/openapi/v1/` for related endpoints; `Proto`'s `.proto` files only if a client needs a new shared message type
- Backend layers: repository/domain, application (command/query handler), API endpoint, for each affected domain
- iOS: the relevant SwiftUI view + view model files in `iOS/`, following that repo's own established conventions (once it has some — this workspace starts empty, so the first few features are where those conventions get set, not just followed)
- Android: the relevant Compose screen + view model files in `Android/`, same note as iOS
- Existing tests for the affected area, in whichever repos are in scope
- Adjacent features that could be impacted — **check both clients once both have real implementations**, since independently-reimplemented business rules on two platforms is the single most common way they silently drift apart over a project's life (true regardless of which specific app this is — see `docs/specs/02-architecture.md`'s "Shared business logic belongs on the backend" section once it exists)

Summarise:
- What the feature requires at each layer, across every repo in scope (not just backend)
- Every file and area that will be touched
- Any ambiguities or gaps in the spec that need clarification before proceeding

Ask any clarifying questions now. Do not proceed to Phase 2 until the user confirms.

---

## Step 6 — Phase 2: Locate & confirm

State exactly which files/areas will change, per repo, based on Step 5's findings. **Wait for explicit user confirmation before any code is written.** This gate stays in this orchestrating session — it needs a live pause for human input, which a delegated agent (see Step 7) cannot do mid-task.

## Step 7 — Phase 3 & 4: delegate implementation to a specialized agent per repo

Once confirmed, do not write the code in this session. Delegate each in-scope repo to its specialized agent via the `Agent` tool:

| Repo in scope | Agent | subagent_type |
|---|---|---|
| `Backend/` | backend-dev | `backend-dev` |
| `iOS/` | ios-dev | `ios-dev` |
| `Android/` | android-dev | `android-dev` |
| Contract (Macro Phase A, `Proto/openapi/v1/`) | — | no dedicated agent yet; handle inline in this session |

**Delegate even when only one repo is in scope.** The point isn't just splitting up multi-repo work — a specialized agent carrying only that repo's conventions (layer order, test patterns, build quirks) consistently produces better-fitted code than this general session attempting every domain. Don't skip delegation for a "small" single-repo fix.

**Sequencing:**
- Macro Phase A (Contract), if required, completes in this session first — small and infrequent enough not to warrant its own agent yet. Don't spawn `backend-dev` until the OpenAPI change is merged.
- **Backend gates iOS/Android only when the feature depends on a new/changed endpoint shape.** In that case, spawn `backend-dev` alone first (foreground — the clients need its exact output), wait for its report, then spawn `ios-dev` and `android-dev` together in a single message once it's back, since neither blocks the other.
- When the feature needs no new/changed backend surface (a client-only UI change), skip straight to that one agent — don't round-trip through backend for nothing.

**Each agent's prompt must be self-contained** — it starts cold, with no memory of this conversation. Include:
- The full spec/issue content (Overview, Business rules, API endpoints/UI interactions)
- The file list this session identified in Step 6, as a starting point, not a constraint — the agent should deepen its own investigation (its own Phase 1) before writing code
- Any locked decisions from an earlier macro phase (e.g. backend's reported endpoint shape, field names, error codes) that this repo's work must match
- An explicit instruction to run that repo's Phase 4 testing checklist below before reporting done, and to author (not publish) that repo's version bump + changelog entry once this workspace's `CLAUDE.md` has defined what that means for this repo (see Phase 5's note — this is one of the things to pin down on the very first feature, not assume)
- An explicit instruction to end with a structured report: files changed, tests run and their result, version bumped to what (once that convention exists), any ambiguity it resolved on its own and why, and each repo's actual merge posture (auto-merge vs. left open for manual merge — confirm which, per repo, the first time this matters, and record the answer in that repo's own `CLAUDE.md` so it's never re-asked)

Wait for each agent's report before treating that repo's phase as done. When relaying context forward to the next agent, pass only what it actually needs (e.g. the new endpoint's request/response shape) — not the full report verbatim.

**Reconcile iOS and Android before moving to Phase 5, whenever both ran.** Once a business rule is implemented on two platforms, compare both agents' reports for any number, boolean, sign, or ordering a user could expect to match across platforms. If one computed something client-side that the other took from the server (or that should come from the server per the "shared business logic belongs on the backend" principle), stop and flag it to the user before Phase 5 — don't let two reports that quietly disagree both get marked done.

---

## Phase 4 — Testing (expanded)

This is the checklist each delegated agent runs for its own repo before reporting back — point it at the matching section in that agent's prompt rather than re-deriving it. Only the sections for repos actually in scope per Step 4 apply. These are starting conventions for a fresh project, not settled ones — the first few features through this workflow are where the *actual* test commands/patterns for each repo get established; update this file once they are, rather than leaving it describing intentions nobody's confirmed yet.

### Backend tests (`Backend/`, C#/.NET)

- Run `dotnet test` (wrap in a `make test` target once this repo has a Makefile, matching the convention below) — all tests must pass with no failures or compilation errors
- For every new/modified command/query handler: write unit tests with mocked repositories (NSubstitute is a reasonable default for C#/xUnit; pick one and stay consistent). Cover: success path, each error/edge case, permission checks
- For every modified handler: update existing tests and add cases for the new behavior
- Target ≥80% coverage on changed files
- Verify `dotnet build -c Release` compiles cleanly with 0 warnings/errors

### iOS tests (`iOS/`, Swift/SwiftUI)

- If the shared-proto message types changed: run whatever this repo's codegen step turns out to be (e.g. `buf generate`) first, so generated types pick up the change
- `xcodebuild build -scheme StreamNext -destination 'platform=iOS Simulator,name=<device>'` (prefix with `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer` if the command-line tools, not full Xcode, are active) — must compile cleanly
- `xcodebuild test -only-testing:StreamNextTests` — all tests must pass. Swift Testing (`@Test`/`#expect`) is a reasonable default; view models tested directly rather than through a mocked network layer is a pattern worth establishing early
- A UI-test smoke suite (XCUITest) for the core user flow is worth having once that flow exists — needs a live local backend; fall back to `build-for-testing` and say so explicitly if no backend is reachable
- SwiftUI view bodies themselves are not worth unit testing (no assertion value without a UI harness) — don't add tests for pure view composition with no new logic
- **Never use screenshots to verify UI behavior** — burns context; use XCUITest assertions instead
- Do a real Simulator visual pass for any UI-visible change before calling it done — build/test passing is not the same as confirming the feature looks and works right

### Android tests (`Android/`, Kotlin/Jetpack Compose)

- `./gradlew test` — JVM unit tests, all must pass with no failures or compilation errors
- For every new/modified ViewModel or use-case/repository class: write unit tests (JUnit5 or Kotest, pick one and stay consistent) with a fake/mocked data layer. Cover: success path, each error/edge case, permission checks
- `./gradlew connectedAndroidTest` (or the Compose UI test equivalent) for any core user-flow change — needs a running emulator/device; fall back to confirming the test target compiles and say so explicitly if none is available
- Compose UI composition itself is not worth unit testing without a semantics-tree assertion (same reasoning as SwiftUI view bodies above) — don't add tests for pure layout changes with no new logic
- `./gradlew lintDebug` (or equivalent) — no new lint errors
- Do a real emulator/device visual pass for any UI-visible change before calling it done

### Translation completeness, iOS (`iOS/StreamNext/Localizable.xcstrings`, once this repo has more than one locale)

If/when this app supports more than one language, apply this checklist — skip it entirely while English is the only supported locale.

- **Compile-time string literals** (`Text("...")`, `Button("...")`, `.navigationTitle("...")`, `Label`, `TextField` placeholders) auto-extract into a String Catalog via `xcodebuild -exportLocalizations` — but extraction adds the key with **no** translation until someone fills it in. Confirm every new/changed key has a value in every supported locale before calling the feature done; a key that exists with no values fails silently at runtime, not at build time.
- **Three traps produce invisible-to-the-catalog text — a recurring Swift/SwiftUI gotcha, not specific to any one app:**
  1. An enum/helper returning a plain `String` from a `switch` — wrap each case in `String(localized: "...", locale: <your app's current-locale source>)`, with an explicit `locale:` (these run outside SwiftUI's environment).
  2. A `String`-typed parameter threaded into `Text(...)`/`Button(...)` from a call site that passes a literal — retype the parameter `LocalizedStringKey` so the existing literal converts implicitly.
  3. A ternary of string literals inside `Text(cond ? "A" : "B")` — the ternary resolves to a `String` before `Text` sees it; move the ternary outside (`cond ? Text("A") : Text("B")`).
- After adding/editing strings, confirm the build still succeeds and spot-check the new keys exist for every locale directly in the catalog file (it's JSON).

### Translation completeness, Android (`Android/.../res/values*/strings.xml`, once this repo has more than one locale)

Same gating note as iOS — skip entirely while English is the only supported locale.

- Every user-visible string goes through `stringResource(R.string.key)` (Compose) or `getString(R.string.key)`, never a hardcoded literal — this is what makes a locale resource folder (`values-es/`, etc.) actually take effect.
- Every locale's `strings.xml` must have the same set of keys as the default `values/strings.xml`. A missing key falls back to the default resource silently — no build error, no runtime crash, just untranslated text, so this needs an explicit check (a small test walking all `values-*/strings.xml` files and diffing keys against the default, mirroring the equivalent test worth having on any multi-locale client), not a visual spot-check.
- After adding translations, confirm the build still passes.

---

## Phase 5 — Document (expanded)

Once every in-scope repo agent has reported back, delegate this phase to the `spec-issues` agent in one pass — hand it every repo's final report (files changed, decisions, version numbers) rather than re-deriving them in this session. It owns the two parts below plus the GitHub issue end-state. This orchestrating session does a final verification (`gh issue view ...`) before telling the user the feature is done — see "Before finishing" below.

### Feature spec (`docs/features/<feature>.md`)

- Set `## Status` to `implemented`
- Add or update the `## Implementation notes` section with: key files changed, patterns used, any non-obvious decisions
- Append a dated entry to `## Change log`: `- YYYY-MM-DD: <what changed> (@mauro)`

### Project specs (`docs/specs/`)

Update any spec file whose content was invalidated by this feature. Specifically:

- **`00-overview.md`** — update Current status if the feature changes what is live
- **`01-requirements.md`** — move the feature from Planned → Implemented (or add it if new); update the description to reflect what was built
- **`02-architecture.md`** — update if the feature introduced a new tech, pattern, or constraint
- **`03-data-model.md`** — update if any of the following changed:
  - A table was created, altered, or dropped (add to a Migration history table, once one exists)
  - A column was added or removed from an entity
  - A new enum value was added to a shared contract type
  - ID type conventions changed

Do not update spec files if nothing in them was affected. Do update them if any row, column, enum, or constraint is now incorrect — stale specs are worse than no specs.

### Version bumps and release notes

**This is undefined for a fresh workspace and needs deciding on the first feature that ships, not assumed from another project.** Once decided (per repo: which file holds the version, patch/minor/major convention, whether release notes are JSON files authored-then-published or something else), write it down in this workspace's root `CLAUDE.md` under a "Version bump" section — mirroring the shape of a `## Version bump` table, if that's the pattern that ends up making sense here — so every future `/implement` run reads it from there instead of re-deciding it. Until that exists, this phase is a no-op beyond flagging that it's pending.

The one rule worth setting from day one regardless of the specifics: **author release notes as part of the feature, do not publish them in the same change.** Publishing announces a build to real users/a real changelog database; that should be a deliberate, separate act at actual release time, never bundled automatically into a feature PR.

### GitHub issue (if linked)

After all spec and docs updates are complete, handle the linked issue.

**GitHub issue path** (argument was `#N` or `N`): the issue number was recorded in Step 1.

**Local file path**: check the spec file for a `## GitHub issue` section. If it contains a value (not just the `<!-- -->` comment placeholder), parse it:
- `#42` or `42` → issue 42 in `StreamNext/StreamNext`
- Full URL `https://github.com/owner/repo/issues/42` → use that repo and number

**Every issue must end this phase either closed, or closed as soon as the last PR merges — never left open with an `in-progress` label.** Issues live in `StreamNext/StreamNext` while all the code lives in sub-repos, which makes the closing mechanism non-obvious. Get it right, or verify by hand.

**A closing keyword in a PR *body* does not work across repositories.** GitHub links a PR to an issue by keyword only within the same repo, so `Closes https://github.com/StreamNext/StreamNext/issues/<n>` in a `Backend` PR description does nothing.

**A closing keyword in a *commit message* does work across repositories**, via the full-URL form, once that commit lands on the sub-repo's default branch. So:

- Put the closing line in the **commit message** of the repo that lands **last** — the one whose merge completes the feature:
  ```
  Closes https://github.com/StreamNext/StreamNext/issues/<number>
  ```
  A bare `(#<number>)` in a sub-repo commit is **not** enough: it resolves to an issue of that number in *that* sub-repo, not the workspace one.
- Earlier repos in the sequence use `Refs <full URL>` instead, so the feature isn't closed while later phases are still outstanding.
- PR bodies should still carry `Refs <full URL>` for traceability, but never rely on a PR body to close anything.

**Confirm each sub-repo's actual merge posture (auto-merge vs. manual) the first time it matters**, and record the answer in that repo's own `CLAUDE.md` — don't assume one way for both iOS and Android just because that happened to be the split in a previous project. If a repo turns out to have no auto-merge, the issue only closes once the user merges that PR by hand when it's last in sequence — say so explicitly in the final report, and check back with `gh issue view <number> --repo StreamNext/StreamNext --json state` if the user mentions merging it.

**Before finishing any `/implement` run, verify the end state:**
```bash
gh issue view <number> --repo StreamNext/StreamNext --json state,labels
```
If it's still `OPEN` and every PR is merged, close it manually with a comment summarising what shipped and any behaviour change worth reviewing:
```bash
gh issue close <number> --repo StreamNext/StreamNext --comment "..."
```

Remove the in-progress label once the issue is closed, whichever way it closed:
```bash
gh issue edit <number> --repo StreamNext/StreamNext --remove-label "in-progress"
```
If the remove fails, log and continue. An issue that is closed but still labelled `in-progress` is the same reporting bug in a quieter form.

### GitHub issue path — create local spec

When the input was a GitHub issue, create `docs/features/<derived-filename>.md` at the end of Phase 5 with `## Status: implemented`, the issue body as the Overview, and the implementation notes and change log filled in. This gives a permanent local record matching the local-file workflow.
