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
| Architecture | TBD | CQRS + MediatR is a reasonable default (see `backend-dev` agent); not yet confirmed as this project's actual convention |
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

See the `backend-dev` agent (`.claude/agents/backend-dev.md`) for the current suggested layer order — this section should be filled in with the *confirmed* convention once the first feature has gone through it, rather than duplicating a still-tentative suggestion here.

## Key backend constraints

<!-- Fill in as real constraints get discovered/decided — e.g. a database connection
     pooling gotcha, an auth token lifetime decision, a rate-limit. Empty is honest
     for a repo with no code yet; don't invent constraints that don't exist. -->

## Key client constraints

<!-- Same note — fill in as iOS/Android conventions get established. -->

## Shared business logic belongs on the backend

If a calculation, filter, or business rule needs to produce the *same answer* on every client (iOS and Android), compute it once in a backend service method and expose it via a REST endpoint — don't let each client re-derive it independently. Independently-reimplemented logic drifts; this is true of every multi-client project, not something specific to this one, so it's worth stating here from day one rather than learning it the hard way once both clients exist.

This does not apply to pure display/presentation logic (row coloring, sort-order-for-a-chart, which fields to show expanded) — only to numbers or booleans a user could reasonably expect to match across screens or platforms. A *display convention* (e.g. how a negative amount is signed/colored) still has to match across clients even though presentation itself is exempt — put the convention in a pure helper on each client, mirrored deliberately and named so the pairing is obvious, and note any intended difference in a doc comment rather than leaving it to be discovered as a bug.

## Feature development process

See `CLAUDE.md` for the full 5-phase workflow, and `.claude/commands/implement.md` for the expanded macro-phase sequencing (Contract → Backend → iOS/Android, independently) and per-repo agent delegation.
