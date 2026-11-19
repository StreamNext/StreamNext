# Feature: Auth

## Status
implemented

## GitHub issue
<!-- Optional -->

## Overview
M1. Accounts and sessions for StreamNext: email + password, Sign in with Apple, and Google sign-in. A session is a short-lived access JWT plus a rotating, device-bound refresh token. Also covers email verification, password reset, the profile (`/v1/me`), FCM device-token registration, and account deletion. FaceID is client-side only: the iOS app keeps the refresh token in the Keychain behind `.userPresence` and does one silent refresh on a 401. The server needs no biometric code.

Local-only: everything here is built and tested without GCP, Resend, Apple or Google credentials. Email goes through a `Mailer` interface and social sign-in through a `TokenVerifier` interface, with fakes in tests and a log-only mailer for local runs.

## Business rules
**Tokens**
- Access token: JWT (HS256, secret `JWT_SECRET`, at least 32 bytes), 15 minutes, claims `sub` (user id), `iat`, `exp`. Nothing else.
- Refresh token: 32 random bytes, base64url, **stored only as a SHA-256 hash**. Lifetime 24h by default, 90 days with `rememberMe` and always for Apple/Google. Each refresh keeps the same lifetime, so the window slides.
- Bound to a `deviceId` (a UUID the client generates). A token presented with a different `deviceId` gets the same 401 as an unknown token.
- Login, register and social sign-in revoke any active refresh token for the same user and device, so a device has one live token.
- **Rotation:** every refresh revokes the presented token and issues a new pair; the old row records `replaced_by`.
- **Reuse detection:** presenting a token that was already **rotated** (it has a successor) revokes *every* refresh token of that user (assume a copy leaked) and returns 401. A token revoked any other way (logout, a newer login on the same device, a password change or reset, or an earlier reuse revocation) is just a 401 with no side effects. *Refined during slice 2: the original wording said any revoked token, which would have let a stale device undo a fresh login after a password reset.*
- **Grace window (10s):** if the presented token was rotated under 10 seconds ago and its successor has never been used, the server revokes the successor and issues a fresh pair, so a client that lost the response can retry. Otherwise it is reuse. *(Not in WellSpent: its docs list this as a gap.)*
- Changing or resetting the password, and deleting the account, revoke all refresh tokens.

**Passwords and accounts**
- Passwords: 10–128 characters, no composition rules. Hashed with argon2id (OWASP minimum: 19 MiB, 2 iterations, 1 lane), stored as a PHC string. No 72-byte limit as with bcrypt.
- Email is trimmed and compared case-insensitively (`citext`). Display name 1–40 characters, defaulting to the provider name or the part of the email before the `@`. Timezone is a valid IANA name, default `UTC`.
- Login failure is always the same 401 `unauthenticated` "invalid credentials", whether the email is unknown or the password is wrong. For an unknown email the server still performs a dummy hash, so response time doesn't reveal which accounts exist.
- Register with an email already in use returns **409 conflict**. This lets a caller learn that an email is registered; accepted for a better sign-up experience, limited by rate limiting.
- Users who signed up with Apple/Google have no password (`password_hash` null) and an already verified email. `change-password` for such an account is 400 `invalid_argument` "account has no password"; setting a first password belongs with slice 3.

**Email verification and reset**
- Register sends a verification email. **Unverified users can log in and track shows**; features that publish content (posting in threads, M5) will require a verified email.
- `email_token`: 32 random bytes, stored hashed, single-use. Verification tokens last 24h, reset tokens 1h. A newly issued token of the same purpose invalidates older unused ones.
- `forgot-password` and `resend-verification` always return 202 with no body difference, whether or not the email exists.
- **Links:** the email carries `{APP_BASE_URL}/verify-email?token=<token>` or `{APP_BASE_URL}/reset-password?token=<token>`, with the token URL-safe base64. This format is the contract the Web repo implements later (see Clients). Plain-text emails only for now.
- **Cooldown:** at most one email per address per purpose per 60 seconds, decided from `email_token.created_at`. A request inside the cooldown still returns 202 and sends nothing, so one address can't be mail-bombed even by many IPs.
- Emails are sent through the `Mailer` interface. `MAIL_PROVIDER=log` (local default) only logs and the server refuses to start with it when `ENV=prod`; `resend` posts to the Resend HTTP API using the standard library.

**Social sign-in** (`POST /v1/auth/apple`, `/v1/auth/google`)
- The client sends the provider ID token. The server verifies the signature (RS256 only) against the provider's published keys (cached 1h, refetched once on an unknown key id, stale keys kept if the provider is briefly unreachable), `iss`, `aud` (Apple: `APPLE_CLIENT_ID`; Google: any of `GOOGLE_CLIENT_IDS`), `exp`, and for Apple the nonce: the client sends the raw nonce and the server compares its SHA-256 (hex) with the token's `nonce` claim.
- A provider whose client id is not configured answers 503 `unavailable`; an invalid or expired token is 401 `unauthenticated`; the provider's key server being down is 503.
- Provider identity is `(provider, subject)`. A known identity signs in its user, whatever the email says now. For an unknown identity:
  - **No email in the token** → 401 (we cannot create an account without one).
  - **Provider-verified email, and a user with that email exists** → the identity is **linked** to that user and their email is marked verified. *Pre-hijacking defense:* if that account's email was never verified, anyone could have registered it with a password they know, so linking first **clears its password and revokes all its sessions**. A verified account keeps its password.
  - **Provider-verified email, no such user** → a new user is created with a verified email and no password.
  - **Unverified provider email** never links: if it matches an existing user the answer is 409 `conflict`; otherwise the user is created unverified and gets the normal verification email.
- New users: display name from the Google `name` claim, else the request's `displayName` (Apple shares the name only on the first authorization, via the client), else the part of the email before the `@`, cut to 40 characters. Timezone `UTC`. The session is device-bound with a 90-day refresh lifetime.
- Two simultaneous first sign-ins of the same identity are handled by re-reading once after a uniqueness conflict. Apple's private-relay addresses are treated as ordinary emails.
- An account that signed in only with a provider has no password. `forgot-password` **does** email them a reset link (the email proves ownership), which is how they set a first password; `change-password` stays 400 because it needs a current password.

**Devices** (`PUT`/`DELETE /v1/devices/{deviceId}`)
- `PUT` upserts the push (FCM) token for `(user, deviceId)`: `fcmToken` 1–4096 characters, `platform` `ios` or `android`; 204.
- An FCM token belongs to exactly one device row. If another user registers the same token (someone else signed in on that device), it **moves** to them, so notifications for the previous user can no longer reach it.
- `DELETE` is idempotent (204 whether or not the row exists). Rows vanish with their user.
- *For M4:* pushes are sent only to devices that also have a live refresh token for that user and device, so logging out stops pushes without a cleanup step.

**Access control**
- Endpoints marked with `bearerAuth` in the contract require `Authorization: Bearer <access token>`; a missing, malformed or expired token is 401 `unauthenticated`. The middleware puts the user id in the request context.
- Rate limiting: a global per-IP limit (10 requests/s, burst 30) and a stricter limit on register, login, refresh, forgot-password, resend-verification and the social endpoints. Excess requests get 429 `rate_limited` with `Retry-After`. *shortcut: in-memory per instance, so with several Cloud Run instances the real limit is higher; move to Postgres/Redis if abused.*
- `DELETE /v1/me` hard-deletes the user, with dependent rows removed by `ON DELETE CASCADE`. Password users must send their password; social-only users just need a valid access token.

**Config added** (all via sops/env, none required in `Config` until used): `JWT_SECRET` (slice 1), `APP_BASE_URL` (default `http://localhost:3000`), `MAIL_PROVIDER`, `RESEND_API_KEY`, `MAIL_FROM`, `ENV` (slice 2); `APPLE_CLIENT_ID`, `GOOGLE_CLIENT_IDS` (comma-separated: iOS, Android, web) (slice 3).

## Clients
- Backend + Proto only for now. iOS and Android follow later (Keychain storage with FaceID, deviceId generation, silent refresh on 401).
- **Web (later, `StreamNext/Web`, currently empty):** two pages the email links point at. `/verify-email?token=` calls `POST /v1/auth/verify-email` and shows the result; `/reset-password?token=` is a new-password form calling `POST /v1/auth/reset-password`. Until they exist the links resolve to nothing, which is acceptable pre-launch (the log mailer prints the token). Open question for that milestone: call the API from Next.js route handlers (no CORS needed) or from the browser (API needs CORS for the Vercel origin).

## API endpoints
Public (no bearer token):

| Endpoint | Purpose | Notable errors |
|---|---|---|
| `POST /v1/auth/register` `{email, password, displayName?, timezone?, deviceId, rememberMe?}` | create account, returns `{accessToken, refreshToken, user}` | 400, 409, 429 |
| `POST /v1/auth/login` `{email, password, deviceId, rememberMe?}` | returns the same session body | 401, 429 |
| `POST /v1/auth/refresh` `{refreshToken, deviceId}` | rotate, returns the same session body | 401 (also on reuse), 429 |
| `POST /v1/auth/logout` `{refreshToken}` | revoke that token; always 200, idempotent | — |
| `POST /v1/auth/apple` `{identityToken, nonce, deviceId, displayName?}` | sign in / sign up, returns a session | 401, 409, 429, 503 |
| `POST /v1/auth/google` `{idToken, deviceId}` | sign in / sign up, returns a session | 401, 409, 429, 503 |
| `POST /v1/auth/verify-email` `{token}` | mark email verified | 400 (invalid or expired token) |
| `POST /v1/auth/resend-verification` `{email}` | always 202 | 429 |
| `POST /v1/auth/forgot-password` `{email}` | always 202 | 429 |
| `POST /v1/auth/reset-password` `{token, newPassword}` | set password, revoke all sessions | 400 |

Authenticated (`bearerAuth`):

| Endpoint | Purpose |
|---|---|
| `GET /v1/me` | profile: id, email, emailVerified, displayName, timezone |
| `PATCH /v1/me` `{displayName?, timezone?}` | update profile |
| `POST /v1/me/change-password` `{currentPassword, newPassword, deviceId, rememberMe?}` | revokes all sessions, returns a new session for that device; wrong current password is 403, an account without a password is 400 |
| `DELETE /v1/me` `{password?}` | delete account: 204; accounts with a password must send it (wrong or missing is 403) |
| `PUT /v1/devices/{deviceId}` `{fcmToken, platform}` | register or update the push token (upsert), 204 |
| `DELETE /v1/devices/{deviceId}` | remove it, 204 (idempotent) |

New error kinds: `TooManyRequests` (429, `rate_limited`). Everything else reuses the existing kinds.

## Data model
Creates `users`, `oauth_account`, `refresh_token`, `email_token`, `device` as sketched in `docs/specs/03-data-model.md`, each in the slice that first uses it: `00002` users + refresh_token (slice 1), `00003` email_token (slice 2), `00004` oauth_account + device (slice 3). Notable constraints: `users.email` citext unique; `oauth_account (provider, subject)` unique; `refresh_token.token_hash` unique, plus a partial unique index on `(user_id, device_id) WHERE revoked_at IS NULL`; `email_token.token_hash` unique; `device (user_id, device_id)` unique. All child tables `ON DELETE CASCADE` from `users`. These are the first sqlc queries, so `sqlc generate` gets wired into `make generate`.

## Testing
- **Service unit tests** (mocked repositories, fake clock and token generator): register (duplicate, weak password, normalization), login (wrong password and unknown email give the same error; the dummy hash runs), refresh rotation, reuse revokes everything, grace window both ways, device mismatch, expiry, remember-me lifetimes, reset and verify tokens (single use, expiry, supersession), social linking rules (verified vs unverified email, new vs existing), delete.
- **Repository tests** (testcontainers Postgres): unique and partial-unique constraints, cascade delete, concurrent refresh of the same token.
- **Handler tests** (httptest): bearer middleware (missing, malformed, expired, valid), rate-limit 429 with `Retry-After`, error body shape.
- **Bruno e2e** (compose stack): register → login → `GET /v1/me` → refresh (rotates) → reuse the old token (401, all sessions revoked) → login again → register device → change password → logout → delete account → `GET /v1/me` 401.
- **Not covered by Bruno:** verify-email and reset (the token only exists in an email), and Apple/Google (need real provider tokens). Those are covered by Go integration tests with a capturing `Mailer` and a fake `TokenVerifier`.
- Coverage gate stays at 90%.

## Slices
Implemented and committed as three reviewable slices, each with its own version bump and Bruno additions:
1. **0.2.0 core (implemented 2026-11-19):** contract + migration, argon2id, JWT, refresh rotation, register/login/refresh/logout, bearer middleware, `GET`/`PATCH /v1/me`, rate limiting.
2. **0.3.0 email flows (implemented 2026-11-19):** `Mailer` (log + Resend), email tokens, verify, resend, forgot/reset, change-password, delete.
3. **0.4.0 social + devices (implemented 2026-11-19):** `TokenVerifier` (Apple, Google), `/auth/apple`, `/auth/google`, device endpoints.

## Decisions (all defaults approved 2026-11-19)
1. Register with a used email: 409 (default) vs. always 202 and email the owner instead.
2. Unverified users can log in (default) vs. blocked until verified.
3. 10-second refresh grace window (default) vs. none, as in WellSpent.
4. In-memory rate limiting per instance (default) vs. a Postgres-backed limiter now.
5. Social-only accounts can delete with just a valid access token (default) vs. requiring a fresh provider sign-in.
6. Three slices (default) vs. one big change.
7. Email token delivery: a link to the Web repo's pages (chosen 2026-11-19) vs. a short code typed in the app. The Web pages are built later, in their own milestone.
8. 60-second per-address email cooldown and plain-text emails (proposed with slice 2).
9. Linking a provider to an unverified password account clears the password and revokes its sessions (pre-hijacking defense); provider-verified linking also verifies the email (approved with slice 3).
10. `forgot-password` works for accounts without a password, so social-only users can set one (approved with slice 3).
11. An FCM token moves to the latest user who registers it; M4 pushes only to devices with a live refresh token (approved with slice 3).

## Implementation notes
**Slice 1 (Backend 0.2.0 / Proto 0.2.0)** — register, login, refresh, logout, `GET`/`PATCH /v1/me`, bearer middleware, rate limiting.

- Layers: `rest` → `service` (interfaces in `service/repositories.go`) → `repository` (sqlc + pgx); `auth` holds the crypto. Migration `00002_auth.sql`; sqlc output is committed.
- Service tests use in-memory fakes that behave like the real repositories (one live token per device, atomic rotate) rather than mocks, plus a simulated race. Postgres-backed tests cover case-insensitive email, the one-live-token index, cascade delete and 8 concurrent rotations (exactly one winner). I mutation-tested the reuse, device-binding, dummy-hash, grace and expiry rules: all five deliberate bugs were caught.
- Bearer protection is read from the contract: the OpenAPI spec is embedded in the generated code and `internal/rest/authmw.go` marks the routes whose `security` is non-empty. A test pins the exact set (`GET`/`PATCH /v1/me`). The generated std-http wrapper does not mark secured operations, so this replaced the original plan.
- Rate limits (constants in `rest.DefaultLimits`): global 10 req/s burst 30 per IP; `/v1/auth/*` additionally 0.5 req/s burst 20. In memory, per instance, keyed on the connection address (see the shortcut comment).
- New config: `JWT_SECRET` (required, at least 32 bytes) and `REFRESH_GRACE_SECONDS` (default 10). Compose sets the grace to 0 and uses a test-only secret, so Bruno can assert reuse detection without a sleep.
- Bruno: 17 auth requests (register, duplicate, weak password, wrong password, unknown email, login, `/me`, 401s, patch, rotation, reuse, logout, refresh after logout). Total suite: 20 requests, 51 assertions.
- Verified: `make test` (97.5% of hand-written code, gate 90%), `make bruno`, and `make migrate`/`make run` against the dev database on the Windows host (migration applied, `/v1/me` rejects missing and bad tokens; no test users created there).
- Behaviours worth knowing: presenting a token that was revoked by **logout** also counts as reuse and revokes every session of that user (as specified); two simultaneous logins on one device can make one fail with a 500 (unique-index violation, rare, not handled); each register/login attempt costs about 19 MiB of memory for argon2id, so the rate limiter is also what protects the server from memory exhaustion (per instance only).
- Not covered by this slice: email verification, password reset, change-password, account deletion (slice 2); Apple/Google and devices (slice 3).

**Slice 2 (Backend 0.3.0 / Proto 0.3.0)** — email verification, password reset, change-password, delete account.

- Layers: `service` gets `Verification`, `PasswordReset` and `Account` (one small file each) over a shared `emailTokens` helper; `internal/mail` composes the emails and delivers through a transport; migration `00003_email_token`. Register calls `Verification.Send` through an optional interface, and a mail failure only logs.
- Clarifications decided while building: a wrong current password is **403** (a 401 would make a client try to refresh a perfectly good access token); `change-password` needs a `deviceId` because it returns a new session; `DELETE /v1/me` answers 204 and its body is optional; an account without a password gets no reset email (slice 3 decides how a social-only user gets a first password); email is plain text and sent inline.
- Rate limits: the strict limiter now also covers `POST /v1/me/change-password` and `DELETE /v1/me`, since a stolen access token could otherwise guess the current password. Defaults are now global 10 req/s burst 60 and strict 0.5 req/s burst 30 (the burst grew so the Bruno suite fits).
- **Behaviour change found by a test:** with bulk revocation (password reset or change), a stale device presenting its old token triggered reuse detection and signed the user out of the session they had just created. Reuse detection now applies only to rotated tokens (see Business rules). Mutation tests cover it.
- Tests: service tests with fakes mirroring the real token rules; Postgres tests for single use, expiry, purpose, supersession, rollback and 8 racing consumers; Resend tested against a fake HTTP server (the real API has never been called: it needs a key and a verified domain). I mutation-tested the cooldown, single-use, weak-password-first, session revocation, password check on delete and no-enumeration rules.
- Verified: `make test` 97.7% of hand-written code, `make bruno` 34 requests / 78 assertions, and the full journey against the dev database with the log mailer (register, verify twice, forgot, weak reset, reset, old/new login, delete wrong/right password, then the deleted account is gone). The test account removed itself.
- Known limits: resetting a password consumes the token before the password is written, so a database failure between the two burns the link (the user asks again); sending inline means response time can hint that an address exists (shortcut comment in `verification.go`); the Web pages the links point to do not exist yet, so the links resolve to nothing until that milestone.

**Slice 3 (Backend 0.4.0 / Proto 0.4.0)** — Apple and Google sign-in, device push tokens.

- Layers: `internal/social` (key cache + token verification, standard library plus the existing JWT package), `service.SocialAuth` and `service.Devices`, repositories for `oauth_account` and `device`, migration `00004`. User creation now takes a `NewUser` struct.
- Rules decided with the slice (see Business rules): pre-hijacking defense on linking, provider-verified linking also verifies the email, an unverified provider email never links, `forgot-password` for passwordless accounts, one owner per FCM token.
- Verifier tests are adversarial and use locally generated RSA keys and a fake key server: forged signature, `alg=none`, HS256 with the public key as secret, RS512, wrong issuer, wrong audience, expired, no expiry, unknown key id, key rotation, rate-limited refetch, provider outage (with and without cached keys), unusable key documents. I mutation-tested the verifier (algorithm list, issuer, audience, nonce comparison, nonce-required, expiry-required, subject, outage classification, string booleans) and the takeover logic (clearing the password, revoking sessions, unverified linking, verified accounts keeping their password, verifying the email, session lifetime, race retry): every deliberate bug was caught. Three mutations first failed to compile and one survived, which exposed that "the app must supply a nonce" was untested; it now is.
- Postgres tests cover single ownership of an identity, token handover between users and between one user's devices, 10 rounds of two users registering the same token at once (the loser retries, up to 3 attempts), and cascades.
- This is the first route with a path parameter, so the generated wrapper's binding-error path is now exercised (a malformed `deviceId` is a 400 through our error handler), and `securedRoutes` is checked against `{deviceId}` route patterns.
- Config: a provider is enabled only when its client id is set; otherwise its endpoint answers 503. The strict rate-limit burst is now 40.
- Verified: `make test` 98.3% of hand-written code, `make bruno` 45 requests / 97 assertions, and against the dev database: migration applied, both providers answer 503 (no client ids in the dev secrets), and two users handed one push token between them. The test accounts deleted themselves.

**What has NOT been verified.** No real Apple or Google token has ever gone through this code: that needs your Apple bundle id / Google client ids and a real device sign-in. The Apple nonce check follows Apple's documented flow (the client sends SHA-256 of a raw nonce to Apple; the server compares the SHA-256 hex of the raw nonce with the token's `nonce` claim). If a real Apple sign-in fails on the nonce, that is the first place to look. Apple's and Google's key endpoints have only been tested against a fake server.

**Known limits and follow-ups**
- Deleting an account does not revoke the Apple token. App Store review requires that for Sign in with Apple users; it needs an Apple developer key, so it is a follow-up before submission.
- There is no way to unlink a provider or list linked providers yet.
- A social-only user sets a first password through `forgot-password`; `change-password` still needs a current password.
- M4 must send pushes only to devices that also have a live refresh token for that user and device (the stale-token rule in Business rules), and must read tokens from the `device` table.
- If the providers' key endpoints are down and nothing is cached, sign-in answers 503 until they recover; each attempt tries to refetch (bounded by the global rate limit).

## Change log
- 2026-11-19: Initial spec (@mauro)
- 2026-11-19: Open decisions 1–6 approved as recommended; tables created per slice
- 2026-11-19: Slice 1 implemented (Backend 0.2.0, Proto 0.2.0)
- 2026-11-19: Slice 2 decisions: links to Web pages (built later), 60s per-address cooldown, social-only change-password rejected
- 2026-11-19: Slice 2 implemented (Backend 0.3.0, Proto 0.3.0); reuse detection narrowed to rotated tokens
- 2026-11-19: Slice 3 implemented (Backend 0.4.0, Proto 0.4.0); feature complete, not yet exercised with real Apple/Google tokens
