# Feature: Auth

## Status
planned

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
- **Reuse detection:** presenting an already-revoked token revokes *every* refresh token of that user (assume theft), returns 401.
- **Grace window (10s):** if the presented token was rotated under 10 seconds ago and its successor has never been used, the server revokes the successor and issues a fresh pair, so a client that lost the response can retry. Otherwise it is reuse. *(Not in WellSpent: its docs list this as a gap.)*
- Changing or resetting the password, and deleting the account, revoke all refresh tokens.

**Passwords and accounts**
- Passwords: 10–128 characters, no composition rules. Hashed with argon2id (OWASP minimum: 19 MiB, 2 iterations, 1 lane), stored as a PHC string. No 72-byte limit as with bcrypt.
- Email is trimmed and compared case-insensitively (`citext`). Display name 1–40 characters, defaulting to the provider name or the part of the email before the `@`. Timezone is a valid IANA name, default `UTC`.
- Login failure is always the same 401 `unauthenticated` "invalid credentials", whether the email is unknown or the password is wrong. For an unknown email the server still performs a dummy hash, so response time doesn't reveal which accounts exist.
- Register with an email already in use returns **409 conflict**. This lets a caller learn that an email is registered; accepted for a better sign-up experience, limited by rate limiting.
- Users who signed up with Apple/Google have no password (`password_hash` null) and an already verified email.

**Email verification and reset**
- Register sends a verification email. **Unverified users can log in and track shows**; features that publish content (posting in threads, M5) will require a verified email.
- `email_token`: 32 random bytes, stored hashed, single-use. Verification tokens last 24h, reset tokens 1h. A newly issued token of the same purpose invalidates older unused ones.
- `forgot-password` and `resend-verification` always return 202 with no body difference, whether or not the email exists.
- Emails are sent through the `Mailer` interface. `MAIL_PROVIDER=log` (local default) only logs and the server refuses to start with it when `ENV=prod`; `resend` posts to the Resend HTTP API using the standard library.

**Social sign-in** (`POST /v1/auth/apple`, `/v1/auth/google`)
- The client sends the provider ID token. The server verifies the signature against the provider's JWKS (cached 1h), `iss`, `aud` (configured client IDs), `exp`, and for Apple the nonce.
- Provider identity is `(provider, subject)`. If it is unknown but the token carries an email the provider marks as verified, and a user with that email already exists, the identity is **linked** to that user. Otherwise a new user is created. An unverified provider email never links.
- Apple's private-relay addresses are treated as ordinary emails.

**Access control**
- Endpoints marked with `bearerAuth` in the contract require `Authorization: Bearer <access token>`; a missing, malformed or expired token is 401 `unauthenticated`. The middleware puts the user id in the request context.
- Rate limiting: a global per-IP limit (10 requests/s, burst 30) and a stricter limit on register, login, refresh, forgot-password, resend-verification and the social endpoints. Excess requests get 429 `rate_limited` with `Retry-After`. *shortcut: in-memory per instance, so with several Cloud Run instances the real limit is higher; move to Postgres/Redis if abused.*
- `DELETE /v1/me` hard-deletes the user, with dependent rows removed by `ON DELETE CASCADE`. Password users must send their password; social-only users just need a valid access token.

**Config added** (all via sops/env, none required in `Config` until used): `JWT_SECRET`, `APPLE_CLIENT_ID`, `GOOGLE_CLIENT_IDS` (comma-separated: iOS, Android, web), `MAIL_PROVIDER`, `RESEND_API_KEY`, `MAIL_FROM`, `ENV`.

## Clients
- Backend + Proto only for now. iOS and Android follow later (Keychain storage with FaceID, deviceId generation, silent refresh on 401).

## API endpoints
Public (no bearer token):

| Endpoint | Purpose | Notable errors |
|---|---|---|
| `POST /v1/auth/register` `{email, password, displayName?, timezone?, deviceId, rememberMe?}` | create account, returns `{accessToken, refreshToken, user}` | 400, 409, 429 |
| `POST /v1/auth/login` `{email, password, deviceId, rememberMe?}` | returns the same session body | 401, 429 |
| `POST /v1/auth/refresh` `{refreshToken, deviceId}` | rotate, returns the same session body | 401 (also on reuse), 429 |
| `POST /v1/auth/logout` `{refreshToken}` | revoke that token; always 200, idempotent | — |
| `POST /v1/auth/apple` `{identityToken, nonce, deviceId, displayName?}` | sign in / sign up | 401, 429 |
| `POST /v1/auth/google` `{idToken, deviceId}` | sign in / sign up | 401, 429 |
| `POST /v1/auth/verify-email` `{token}` | mark email verified | 400 (invalid or expired token) |
| `POST /v1/auth/resend-verification` `{email}` | always 202 | 429 |
| `POST /v1/auth/forgot-password` `{email}` | always 202 | 429 |
| `POST /v1/auth/reset-password` `{token, newPassword}` | set password, revoke all sessions | 400 |

Authenticated (`bearerAuth`):

| Endpoint | Purpose |
|---|---|
| `GET /v1/me` | profile: id, email, emailVerified, displayName, timezone |
| `PATCH /v1/me` `{displayName?, timezone?}` | update profile |
| `POST /v1/me/change-password` `{currentPassword, newPassword}` | revokes all sessions, returns a new session for this device |
| `DELETE /v1/me` `{password?}` | delete account |
| `PUT /v1/devices/{deviceId}` `{fcmToken, platform}` | register or update the push token (upsert) |
| `DELETE /v1/devices/{deviceId}` | remove it |

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
1. **0.2.0 core:** contract + migration, argon2id, JWT, refresh rotation, register/login/refresh/logout, bearer middleware, `GET`/`PATCH /v1/me`, rate limiting.
2. **0.3.0 email flows:** `Mailer` (log + Resend), email tokens, verify, resend, forgot/reset, change-password, delete.
3. **0.4.0 social + devices:** `TokenVerifier` (Apple, Google), `/auth/apple`, `/auth/google`, device endpoints.

## Decisions (all defaults approved 2026-11-19)
1. Register with a used email: 409 (default) vs. always 202 and email the owner instead.
2. Unverified users can log in (default) vs. blocked until verified.
3. 10-second refresh grace window (default) vs. none, as in WellSpent.
4. In-memory rate limiting per instance (default) vs. a Postgres-backed limiter now.
5. Social-only accounts can delete with just a valid access token (default) vs. requiring a fresh provider sign-in.
6. Three slices (default) vs. one big change.

## Implementation notes
<!-- Filled in after implementation (Phase 5) -->

## Change log
- 2026-11-19: Initial spec (@mauro)
- 2026-11-19: Open decisions 1–6 approved as recommended; tables created per slice
