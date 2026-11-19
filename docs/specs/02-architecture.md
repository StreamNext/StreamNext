# Architecture

## System design

Multi-repo, spec-driven — same shape as the workspace tooling itself:

```
StreamNext (workspace root)
  ├── docs/specs/   ← product specs and requirements
  ├── Proto/        ← OpenAPI (REST contract) + protobuf (shared client types, if needed)
  ├── Backend/      ← C#/.NET REST + JWT API server
  ├── iOS/          ← SwiftUI native iOS app (not created yet)
  └── Android/      ← Kotlin/Compose native Android app (not created yet)
```

```
docs/specs/ → Proto (OpenAPI + protobuf) → Backend (C#)
                                                   ↑
                                     iOS / Android (REST calls)
```

## Tech stack

### Backend

| Concern | Choice | Why |
|---|---|---|
| Transport | REST + JWT | Decided — no RPC framework; any client needs nothing special to call it |
| API contract | OpenAPI (`Proto/openapi/v1/`) | Hand-authored, PR-reviewed, consumed by all clients' codegen at build time |
| Language | C#/.NET | Decided |
| Architecture | TBD | CQRS + MediatR is a reasonable default (see "Backend layer order" below); not yet confirmed as this project's actual convention |
| Data access | TBD | EF Core is a reasonable default for C#; not yet confirmed |
| Database | TBD | |
| Deployment | TBD | |

### iOS

SwiftUI, REST via an OpenAPI-generated client against the contract above. Not created yet.

### Android

Kotlin, Jetpack Compose, REST via an OpenAPI-generated client + Retrofit/Ktor against the contract above. Not created yet.

## Transport

Plain REST end to end, by decision — there is no second transport to reconcile. If `Proto` carries `.proto` files, those are shared *message-type* definitions only (for cross-platform codegen convenience), never a live RPC surface — don't add `service`/`rpc` blocks expecting something to implement them.

## Backend layer order

Suggested starting order for a new feature — **tentative**: confirm or adjust it once the first feature has gone through it, and write the confirmed version into `Backend/CLAUDE.md`.

1. **Contract** — the endpoint is defined in `Proto/openapi/v1/` first, hand-authored, not generated from the backend's code.
2. **Domain + Infrastructure** — entity/model, a repository abstraction, and its data-access implementation (EF Core is a reasonable default for C#).
3. **Application** — command/query + handler, with unit tests against a mocked repository (no live DB in unit tests). CQRS (one handler per command/query) is a reasonable default, not mandatory.
4. **Api** — the HTTP endpoint, wired to the application layer.

Decisions worth making on the first feature and writing into `Backend/CLAUDE.md` so they are never re-decided:
- **Errors:** one exception-to-HTTP-status mapping applied from the first endpoint (e.g. typed not-found/forbidden/already-exists/invalid-argument exceptions mapped by one middleware), so every endpoint's error shape is the same.
- **Tests:** one test framework and one mocking library (xUnit and NSubstitute are common defaults). NSubstitute returns `null`, not an empty collection, for an unconfigured `Task<List<T>>` method, so code reading a repo result should defensively `?? []`.
- **Migrations:** EF Core Migrations or hand-written SQL applied by a separate migrator. Pick one.
- **Connections:** a distinct `application_name` (or the DB's equivalent) on every DB connection from day one; cheap now, painful to retrofit.
- **Logging:** every non-trivial path logs with the IDs that matter, and no exception is swallowed silently.
- **Version bump, release notes and git/merge workflow:** undefined until the first feature ships; write them into the repo's `CLAUDE.md` once settled.

## Key backend constraints

<!-- Fill in as real constraints get discovered/decided — e.g. a database connection
     pooling gotcha, an auth token lifetime decision, a rate-limit. Empty is honest
     for a repo with no code yet; don't invent constraints that don't exist. -->

## Key client constraints

<!-- Fill in as iOS/Android conventions get established. The starting points below are tentative, not settled. -->

**Both clients**
- REST against the OpenAPI contract, with a client generated from `Proto/openapi/v1/` rather than hand-written request/response types. If `Proto` also carries shared `.proto` message types, decide once whether the clients use them directly or map REST responses into them, and make iOS and Android match.
- MVVM, one screen and one view model per feature screen, written so a screen's design reads the same on both platforms. Test the view model directly against a fake data layer, not through full integration. Don't unit test pure layout.
- Verify UI with test assertions (XCUITest / Compose test rules), never screenshots, which burn context.
- Version numbers: each platform has a monotonic build number and a user-visible version. Decide one bump convention (build number every feature, user-visible version only at an intentional release) and apply it to both. Mobile repos commonly leave the PR open for a human to merge after a build check rather than auto-merging; confirm per repo.

**iOS (SwiftUI)**
- Build and test with `xcodebuild build|test -scheme StreamNext -destination 'platform=iOS Simulator,name=<device>'`, prefixed with `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer` if it says it "requires Xcode". If the REST client comes from an SPM build plugin (e.g. `swift-openapi-generator`), unattended builds need `-skipPackagePluginValidation`.
- Swift Testing (`@Test`/`#expect`) is a reasonable default. UI smoke tests need a live local backend and test credentials; fall back to `build-for-testing` and say so if none is reachable.
- SPM local packages auto-glob, so a new file in an existing package needs no `project.pbxproj` change. Adding or removing an actual SPM dependency touches five sections (`PBXBuildFile`, Frameworks build phase, per-target `packageProductDependencies`, top-level `packageReferences`, and the `XCRemoteSwiftPackageReference`/`XCSwiftPackageProductDependency` definitions); check with `plutil -lint` and a brace count after each pass.
- Never quit a running Xcode to clear stale state without asking.

**Android (Kotlin, Jetpack Compose)**
- `./gradlew build`, `./gradlew test` (JVM unit tests), `./gradlew connectedAndroidTest` (needs an emulator or device; otherwise confirm the target compiles and say so), `./gradlew lintDebug`.
- `ViewModel` with `StateFlow`/Compose state; one DI approach (Hilt or manual constructor injection) chosen early and used consistently.
- JUnit5 or Kotest for view-model tests; Compose tests (`createComposeRule`, `onNodeWithText(...).assertExists()`) for a core-flow smoke test.
- `versionCode` plays the build-number role and `versionName` the user-visible one.

## Shared business logic belongs on the backend

If a calculation, filter, or business rule needs to produce the *same answer* on every client (iOS and Android), compute it once in a backend service method and expose it via a REST endpoint — don't let each client re-derive it independently. Independently-reimplemented logic drifts; this is true of every multi-client project, not something specific to this one, so it's worth stating here from day one rather than learning it the hard way once both clients exist.

This does not apply to pure display/presentation logic (row coloring, sort-order-for-a-chart, which fields to show expanded) — only to numbers or booleans a user could reasonably expect to match across screens or platforms. A *display convention* (e.g. how a negative amount is signed/colored) still has to match across clients even though presentation itself is exempt — put the convention in a pure helper on each client, mirrored deliberately and named so the pairing is obvious, and note any intended difference in a doc comment rather than leaving it to be discovered as a bug.

## Feature development process

See `CLAUDE.md` for the full 5-phase workflow, and `.claude/commands/implement.md` for the expanded macro-phase sequencing (Contract → Backend → iOS/Android, independently).
