# Architecture

## System design

Multi-repo, spec-driven — same shape as the workspace tooling itself:

```
StreamNext (workspace root)
  ├── docs/specs/   ← product specs and requirements
  ├── Proto/        ← OpenAPI (REST contract) + protobuf shared message types (BSR)
  ├── Backend/      ← Go REST + JWT API server + scheduled jobs
  ├── iOS/          ← SwiftUI native iOS app (not created yet)
  └── Android/      ← Kotlin/Compose native Android app (not created yet)
```

```
docs/specs/ → Proto (OpenAPI + protobuf) → Backend (Go)
                                                ↑
                              iOS / Android / web (REST calls)
```

Runtime:

```
iOS / Android / Vercel web ──REST+JWT──▶ Cloud Run service: api (cmd/server)
                                            │  pgx ▶ Neon Postgres
                                            │  read-through cache ▶ TheTVDB v4
Cloud Scheduler ─▶ Cloud Run Jobs:
   tvdb-sync   (hourly)   /updates diff ▶ upsert ▶ notifications (renewed / ended / new episodes)
   air-alerts  (15 min)   episodes airing soon for followed shows ▶ FCM
```

## Tech stack

### Backend

| Concern | Choice | Why |
|---|---|---|
| Transport | REST + JWT | Decided — no RPC framework; any client needs nothing special to call it |
| API contract | OpenAPI 3.0 (`Proto/openapi/v1/streamnext.yaml`) | Hand-written and PR-reviewed. The backend generates its server interface from it with oapi-codegen (`std-http-server`); clients generate their own. |
| Shared types | protobuf messages/enums (`Proto/streamnext/v1/`) | Messages only, no `service` blocks. Pushed to BSR `buf.build/streamnext/streamnext` for client codegen |
| Language | Go | Decided |
| Architecture | `rest` (handlers) → `service` → `repository` | Services take repository interfaces, so they can be unit tested with mocks |
| Data access | sqlc + pgx v5 | SQL is hand-written and type-checked at generate time |
| Migrations | goose, SQL files in `internal/db/migrations` | Run by `cmd/migrate` before each deploy |
| Database | PostgreSQL 16. **Dev:** a standalone StreamNext container (`Backend/docker-compose.db.yml`, own volume, port 5433) started once on the LAN dev host, separate from WellSpent's Postgres. **Tests:** throwaway containers (testcontainers, compose). **Prod:** Neon (later). | Neon via the **direct** (non-pooler) URL. The dev DB is fully defined in the repo, so it can be recreated from scratch on any machine. |
| Secrets | sops + age, `<env>.enc.env` committed (the `.env` suffix is what lets `sops exec-env` detect the format) | CI decrypts with `SOPS_AGE_KEY`; critical secrets go to GCP Secret Manager |
| Deployment | GCP Cloud Run (service + jobs), Artifact Registry, Cloud Scheduler | GitHub Actions authenticates through Workload Identity Federation |
| Push | Firebase Cloud Messaging, for iOS and Android | One sender; FCM forwards to APNs |
| Email | Resend | Verification and password reset |
| Show data | TheTVDB v4 (licensed project key) | The server caches what it needs; clients never call TheTVDB directly |
| GIFs | GIPHY | The API key stays on the server; we store only GIF ids |
| Testing | `go test` + testify, testcontainers-go (Postgres), Bruno (`bru run`) | Coverage gate at 90% on `internal/` |

### iOS

SwiftUI, REST via an OpenAPI-generated client against the contract above. Not created yet.

### Android

Kotlin, Jetpack Compose, REST via an OpenAPI-generated client + Retrofit/Ktor against the contract above. Not created yet.

## Transport

Plain REST end to end, by decision — there is no second transport to reconcile. `Proto`'s `.proto` files are shared *message-type* definitions only (for cross-platform codegen convenience), never a live RPC surface — don't add `service`/`rpc` blocks expecting something to implement them.

## Backend layer order

Order for a new feature (confirm on the first feature, then record it in `Backend/CLAUDE.md`):

1. **Contract** — define the endpoint in `Proto/openapi/v1/streamnext.yaml` first, by hand; it is never generated from backend code. Regenerate `gen/` in Backend.
2. **Migration + queries** — a goose migration in `internal/db/migrations`, sqlc queries in `internal/db/query`, then `sqlc generate`.
3. **Repository** — an interface the service consumes, implemented over sqlc. Tested against real Postgres with testcontainers.
4. **Service** — the business rules, unit tested with mocked repositories (no live DB).
5. **REST handler** — implements the generated server interface and maps between contract types and service calls. Tested with `httptest`.
6. **Bruno** — add or extend requests in `Backend/bruno/` for the new endpoints.

Decisions recorded up front:
- **Errors:** typed errors in `internal/apperr` (not-found, forbidden, conflict, invalid-argument, unauthenticated), mapped to HTTP status in one place. Error body is lowercase `{"code","message"}` exactly as in the contract.
- **Connections:** every process sets its own `application_name` (`streamnext-api`, `streamnext-tvdb-sync`, …).
- **Logging:** zap, structured. Every non-trivial path logs the IDs that matter, and no error is swallowed silently.
- **Version bump, release notes and git/merge workflow:** decided in M0 and written into `Backend/CLAUDE.md`.

## Key backend constraints

- **TheTVDB licensing:** we have a licensed project key, so the backend may cache TheTVDB data and serve it to all users. Clients never call TheTVDB directly.
- **TheTVDB `/updates`:** `since` must be less than 90 days ago. If the sync cursor is older, re-fetch every tracked series instead.
- **TheTVDB status:** the only values are Continuing, Ended and Upcoming. "Renewed" is derived from a new season record or Ended→Continuing. "Cancelled" can't be told apart from "Ended".
- **Air times:** TheTVDB air times are local to the series' country (its capital or most populous city); for streaming services they are the official release time. Converting to UTC needs a country→timezone map.
- **Auth tokens:**
  - The access JWT (HS256) lasts 15 minutes.
  - The refresh token is opaque, stored as a SHA-256 hash, bound to a `deviceId`, and rotated on every refresh.
  - Reusing a revoked refresh token revokes every token the user has.
  - Refresh lifetime is 24h, or 90d with remember-me and for social sign-in.
  - A short grace window covers a client that lost a rotated refresh response.
- **Neon:** use the direct (non-pooler) URL. pgx and PgBouncer transaction pooling don't mix.

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
