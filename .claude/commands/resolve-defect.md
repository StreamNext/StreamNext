Diagnose and fix a defect using a test-driven, red-green loop: understand → locate → write a failing test → fix → verify.

**Usage:**
- `/resolve-defect <plaintext description>` — a free-text description of what's wrong (symptoms, steps to reproduce, when it started)
- `/resolve-defect #<N>` or `/resolve-defect <N>` — fetch from GitHub issue N in `StreamNext/StreamNext`

**Examples:** `/resolve-defect "login crashes when the token refresh overlaps with a logout"` · `/resolve-defect #94`

This is `/implement`'s sibling for bugs, not new features: same multi-repo awareness, same delegation to specialized agents, same documentation/versioning discipline at the end — but the middle of the workflow is a TDD loop (a failing test proves the bug and later proves the fix) instead of a 5-phase build.

---

The user has run `/resolve-defect $ARGUMENTS`.

If `$ARGUMENTS` is empty, stop and show the usage above.

## Step 0 — Read the project specs

Read all four spec files in parallel before doing anything else, even if you believe you already know the project state:
- `docs/specs/00-overview.md` — what the product is and what repos exist
- `docs/specs/01-requirements.md` — what is implemented vs planned
- `docs/specs/02-architecture.md` — tech stack, layer order, key constraints
- `docs/specs/03-data-model.md` — all entities, columns, ID types, enums, migration history

## Step 1 — Gather the defect description

**Determine input type** from `$ARGUMENTS`:
- A bare number (`3`, `42`) or hash-prefixed number (`#3`, `#42`) → GitHub issue path.
- Otherwise → plaintext path, using `$ARGUMENTS` verbatim as the defect description.

### GitHub issue path

```bash
gh issue view <number> --repo StreamNext/StreamNext --json title,body,labels,type
```

Use the issue **title** + **body** as the defect description. If this org has configured GitHub Issue Types (check with `gh api orgs/StreamNext/issue-types` the first time this matters) and `type` isn't `Bug`, proceed anyway but note the mismatch to the user rather than silently assuming it's wrong. If a `.github/ISSUE_TEMPLATE/bug.md` exists with structured sections (Observed/Expected/Steps to reproduce), prefer those over raw title+body.

Mark the issue as in-progress, same fallback as `/implement`:
```bash
gh issue edit <number> --repo StreamNext/StreamNext --add-label "in-progress"
```
If this fails because the label doesn't exist, create it (`gh label create "in-progress" --repo StreamNext/StreamNext --color "fbca04" --description "Work is underway"`) then retry. If it still fails, log and continue.

If `gh issue view` fails, stop and report the error.

### Plaintext path

No issue to fetch or label. Record the description as-is; there is no "in-progress" marker to set since there's no durable record yet — Phase 6 is where this gets written down.

## Step 2 — Determine scope

A defect almost never needs a contract change — the contract is usually correct and the *implementation* of it is wrong. Only scope **Macro Phase A (Contract)** if Phase 2's investigation below genuinely finds the wire contract itself is the defect (a wrong field type, a missing enum value); don't assume it by default the way a new feature might.

Determine which repo(s) are actually implicated — **most defects live in exactly one repo.** Don't assume multi-repo scope the way a cross-cutting feature might; only treat it as multi-repo if the investigation in Phase 2 actually finds the same bug duplicated in more than one codebase (once both iOS and Android have real implementations, this is the most common way a defect turns out to be two-repo: the same business rule reimplemented and drifted apart on each platform — see `docs/specs/02-architecture.md`'s "Shared business logic belongs on the backend" section, once it exists).

## Phase 1 — Understand

Read the defect description in full. Before investigating code, make sure you actually know:
- What the observed (wrong) behavior is, concretely — not just "X is broken"
- What the expected (correct) behavior should be instead
- Reproduction conditions: how reliably, under what data/state, since when

If any of these are missing or ambiguous, ask the user now — use `AskUserQuestion` for concrete, narrow questions (e.g. "does this happen every time or only sometimes?", "on which platform?") rather than guessing. A wrong assumption here wastes the entire loop below. Don't proceed to Phase 2 until the defect itself is unambiguous.

## Phase 2 — Locate & confirm

Investigate the actual code to find where the defect lives, across whichever layers/repos are plausible given Phase 1 (shared contract types, backend domain/application/API layers, iOS view/view model, Android screen/view model). Ground every claim in code you actually read — not in what a doc *says* the code does, since docs drift.

**If more than one plausible root cause exists, say so and narrow it with evidence** — read the actual data/logs/state if the report provides them, or ask the user for a concrete example (an ID, a screenshot of values, a query result) rather than committing to the first hypothesis that fits the symptoms. A hypothesis that merely *could* explain the symptom is not the same as one *confirmed* against real data.

**A real possible outcome of this phase is "there is no code defect here."** Sometimes the report is a data problem (bad/duplicate rows), a configuration issue, or expected behavior the user didn't anticipate. If investigation leads there, stop and say so plainly instead of forcing Phase 3 to invent a test for a bug that doesn't exist in code — that's a different conversation (data cleanup, a product decision, user education), not a TDD loop.

Once you have a confirmed root cause: **state exactly which file(s)/function(s) are wrong and why, and wait for explicit user confirmation before writing any test or code.** This gate stays in this orchestrating session — a delegated agent can't pause mid-task for human input.

## Phase 3 — Red: write a failing test

Once confirmed, delegate the rest (Phases 3–5, the actual TDD loop) to the specialized agent for the repo in scope:

| Repo in scope | Agent |
|---|---|
| `Backend/` | `backend-dev` |
| `iOS/` | `ios-dev` |
| `Android/` | `android-dev` |

**Delegate even for a one-line fix** — same reasoning as `/implement`: a specialized agent carrying that repo's test conventions and build quirks does this more reliably than a general session. If more than one repo is genuinely in scope (per Step 2), spawn their agents together in one message — neither blocks the other for a defect fix the way backend can gate a client for a new endpoint in a feature.

**Each agent's prompt must be self-contained** (it starts cold) and must instruct it to run this exact loop, not just "fix the bug":

1. Write one or more tests that encode the **correct** expected behavior, following that repo's existing test conventions and patterns for the affected area (or establishing a sensible one if none exists yet for this area — this is a fresh codebase).
2. Run the test(s) and **confirm they fail** before touching any fix. A test that passes immediately against unfixed code doesn't capture the defect — revise the test (or, if no revision makes it fail, stop and report that the diagnosis may be wrong rather than forcing a fix with no test backing it).
3. **Apply the fix** — the smallest change that makes the test pass. Don't refactor unrelated code or add speculative abstractions while in here; a defect fix is not a cleanup pass.
4. Re-run the new test(s): must now pass (green).
5. Run that repo's **full** existing test suite (not just the new test) to confirm no regression. If the full suite breaks something else, the fix's scope is probably wrong — reconsider Phase 4, don't just patch around the new failure.
6. **If the new test still fails after the fix, return to writing the test** — reconsider whether it actually encodes the expected behavior and whether the diagnosis (the file/function identified in Phase 2) was correct; adjust the test and/or the fix, then re-run. Loop until genuinely green, not until the test is weakened enough to pass.

Also instruct the agent to: author (not publish) that repo's version bump + changelog entry once this workspace has defined what that means for this repo (see `/implement`'s Phase 5 note on this — it's undefined until the first feature through that workflow pins it down), and end with a structured report — files changed, the test(s) added and their before/after result, the fix applied, full-suite result, version bumped to what (once that convention exists).

## Phase 4 — Green: the fix

Covered inside the Phase 3 delegation above — the agent applies the fix as step 3 of that same loop. Nothing additional happens in this orchestrating session; this phase exists as a heading only so the loop's five named steps are traceable 1:1 against this file.

## Phase 5 — Verify

Also covered inside the Phase 3 delegation (steps 4–6). Wait for each agent's report before treating that repo's phase as done.

**If both iOS and Android were in scope, reconcile their reports** before moving on — same check `/implement` runs: compare any number, boolean, sign, or ordering a user could expect to match across platforms, and flag it before Phase 6 if they quietly disagree.

## Phase 6 — Document & close

Once every in-scope agent has reported back:

### Feature doc, if this defect belongs to one

Check whether the fixed area has an existing `docs/features/<feature>.md`. If so, append a dated `## Change log` entry describing the symptom and the fix (same format every other entry in that file uses: `- YYYY-MM-DD: <what was wrong, root cause, what changed> (@mauro)`). Don't create a new feature spec file for a defect — that's what `/spec` is for, and a bug fix isn't a new feature. If the defect doesn't map cleanly to any single existing feature doc, skip this rather than forcing a fit.

### Version bumps and release notes

See `/implement`'s Phase 5 note — this workspace hasn't pinned down per-repo version/changelog conventions yet. Once it has (recorded in the root `CLAUDE.md`), follow it here too. Author the notes; never publish them from here.

### GitHub issue (if this was the issue path)

Same mechanics as `/implement`'s Phase 5 — summarized here, not re-derived:
- A closing keyword in a PR **body** does nothing cross-repo. Put `Closes https://github.com/StreamNext/StreamNext/issues/<number>` in the **commit message** of whichever repo's change lands last; earlier repos use `Refs <full URL>` instead.
- Confirm each sub-repo's actual merge posture (auto-merge vs. manual) before assuming — if the repo that lands last has no auto-merge, the issue only closes once the user merges that PR. Say so, and check back with `gh issue view <number> --repo StreamNext/StreamNext --json state` if the user mentions merging it.
- Before finishing, verify: `gh issue view <number> --repo StreamNext/StreamNext --json state,labels`. If still `OPEN` with every PR merged, close it by hand with a comment summarizing the root cause and fix, and remove the `in-progress` label regardless of how it closed.

### Plaintext path — no issue to close

Just confirm with the user that the reported symptom is actually gone (re-describe what changed in plain terms), since there's no issue thread to serve as that confirmation.
