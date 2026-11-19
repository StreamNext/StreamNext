# Feature: Foundation

## Status
implemented

## GitHub issue
<!-- Optional -->

## Overview
M0. This builds the skeleton every later milestone plugs into: a Go backend that boots, reads config and secrets, migrates Postgres and serves one endpoint from the OpenAPI contract. It is tested and deployable, with CI enforcing coverage and running the Bruno integration suite. Proto gets its contract file, a buf module, and CI that lints and pushes to BSR.

## Business rules
- **Contract first:** `GET /v1/health` and the shared `Error` schema are defined in `Proto/openapi/v1/streamnext.yaml`. Backend generates its server interface with oapi-codegen (`std-http-server`) into gitignored `gen/`.
- **Health:** `GET /v1/health` returns `200 {"status":"ok","version":"<x.y.z>"}` when Postgres answers a ping, and `503` with the standard error body otherwise. It needs no auth.
- **Errors:** every non-2xx response has the body `{"code","message"}` (lowercase). Typed errors in `internal/apperr` map to HTTP status in one place. Panics are recovered as `500 internal` and logged.
- **Config:** environment variables only. Locally they are loaded from a decrypted `<ENV>.env`; committed copies are sops-encrypted `<ENV>.enc.env` with a **new StreamNext age key**, not WellSpent's. Files are named `<env>.enc.env`: the `.env` suffix is what lets `sops exec-env` detect the format (`.env.<env>.enc` made it fail to parse). The process fails fast at startup if a required variable is missing.
- **Database:** pgx v5 pool, with `application_name` set per process (`streamnext-api`, `streamnext-migrate`). goose migrations are embedded in the binary and run by `cmd/migrate`. `sqlc.yaml` is configured, but `sqlc generate` refuses to run with zero queries, so generation is wired in with the first real query (M1).
- **Tools pinned in `go.mod`** (`go tool goose`, `go tool oapi-codegen`), so no global installs are needed. Bruno and Redocly run through `npx` at pinned versions.
- **Coverage gate:** CI fails if coverage of `internal/` (excluding `gen/` and `internal/sqlc`) is below 90%.
- **Bruno:** `Backend/bruno/` has a `local` and a `ci` environment and a `health` request with assertions. CI starts Postgres and the API with `docker compose`, then runs `bru run --env ci`.
- **Deploy:** Dockerfile (distroless) builds `server` and `migrate`. A manual run (`workflow_dispatch`; automatic push-to-`main` is disabled until GCP is signed off) builds, pushes to Artifact Registry, runs migrations and deploys the Cloud Run service. GitHub authenticates to GCP through Workload Identity Federation (no JSON key).
- **Git workflow (Backend and Proto):** work on `develop`, open a PR to `main`, auto-merge when CI is green. No AI attribution trailers.
- **Versioning (decided here per workspace CLAUDE.md):** the version lives in `internal/version/version.go` (Backend) and in the OpenAPI `info.version` (Proto). Semver: bump minor for a new feature or endpoint, patch for a fix, major for a breaking contract change. Release notes are one dated entry per version in each repo's `CHANGELOG.md`.

## Clients
- None (Backend + Proto only)

## API endpoints
- `GET /v1/health` — liveness + DB check, returns status and server version

## Implementation notes
- Backend 0.1.0 / Proto 0.1.0, committed locally on `develop` in each repo (not pushed yet).
- `internal/{apperr,config,db,rest,version}` with 96.2% coverage; db tests run a shared Postgres 16 testcontainer (~2s). `cmd/` is outside the gate and kept thin.
- Verified locally: `make test`, `make bruno` (compose: Postgres → migrate → API → Bruno, 3 assertions), and `make migrate` / `make run` through sops with the new age key (`~/Library/Application Support/sops/age/streamnext.txt`).
- Proto: `buf lint` / `buf format` and Redocly lint pass. Proto CI's BSR push needs the `BUF_TOKEN` repo secret.
- Dev database (added 2026-11-19): `Backend/docker-compose.db.yml` defines a standalone StreamNext Postgres (user `streamnext`, db `streamnext_dev`, port 5433, own volume). `make db-up` writes `.env.db` from the sops secrets and starts it. Verified from scratch twice on a wiped volume (container up, migrate, `citext` present). `dev.enc.env` points `DATABASE_URL` at `192.168.68.52:5433`, an assumption: change the host if the DB runs elsewhere. Not yet started on that host.
- **GCP deployment is on hold** until the user signs off; `deploy.yml` is manual-trigger only.
- **Not verified:** GitHub Actions (`ci.yml`, `deploy.yml`) have never run, and `deploy.yml` also needs the GCP project, Workload Identity Federation, Artifact Registry repo, and Secret Manager secret `streamnext-database-url`.
- The `ErrorHandlerFunc` branch in `rest.Handler` (parameter-binding errors) has no test yet: no endpoint has parameters until M1.
- Versioning and git workflow are recorded in `Backend/CLAUDE.md` and `Proto/CLAUDE.md`. Proto CI uses `bufbuild/buf-action`; breaking-change checks are off until the first release.

## Change log
- 2026-11-19: Initial spec (@mauro)
- 2026-11-19: Implemented (Backend 0.1.0, Proto 0.1.0)
