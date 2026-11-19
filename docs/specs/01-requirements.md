# Functional Requirements

## Implemented

### M0 — Foundation
Go backend skeleton with `GET /v1/health`, sops secrets, goose migrations, 90% coverage gate and a Bruno suite; Proto contract with buf module. Implemented locally; CI and deploy are unverified.
→ [Feature spec](features/foundation.md)

<!-- Each further entry, once a feature ships, should look like:

### Feature name
One or two sentences describing what's implemented and its key business rule.
→ [Feature spec](features/feature-name.md)
-->

## Planned

Backend milestones, in build order. Each gets a `docs/features/*.md` spec via `/spec` right before it is implemented.

### M1 — Auth *(slices 1 and 2 of 3 implemented: sessions, `/me`, email verification, password reset, change-password, delete account; slice 3 is Apple/Google and devices; see [auth.md](../features/auth.md))*
Email and password (verification and reset through Resend), Sign in with Apple, and Google sign-in, where the server verifies the native SDK's ID token. Access JWTs last 15 minutes. Refresh tokens are opaque, bound to a device and rotated on every use; reusing a revoked one revokes every token the user has. FaceID is client-side, guarding the refresh token in the Keychain. Also covers FCM device-token registration and account deletion.

### M2 — Catalog
Show search (TheTVDB proxy) and series detail with seasons and episodes, read through and cached in Postgres.

### M3 — Tracking
Follow and unfollow shows with a list state (watching, planned, paused, dropped). Mark an episode, a season, or everything up to an episode as watched, and unwatch. "Up next" per show and across all shows, plus an upcoming-episodes calendar.

### M4 — Sync + notifications
An hourly job applies TheTVDB `/updates` to cached shows and raises **renewed** (a new season, or Ended→Continuing) and **ended** (Continuing→Ended) events. TheTVDB has no separate "cancelled" status. A 15-minute job sends airing-soon alerts. Push goes through FCM on both platforms, with an in-app inbox and per-user and per-show preferences.

### M5 — Episode threads
Posts with one level of replies, reactions, edit and delete, and reports (a post is auto-hidden after N reports). GIFs come from GIPHY. A thread is readable and postable only after the user has watched the episode or unlocked spoilers for that one episode; the unlock is remembered.

### M6 — Analytics
Total shows, episodes and minutes watched; completed shows; a monthly trend; top genres and networks.

### M7 — Discovery
Similar shows and "for you" picks, scored by genre overlap, shared network, and how often other users follow both shows.

### M8 — Import from other trackers *(placeholder)*
Import follows and watch history from TV Time and Trakt, mapped to TheTVDB ids. → [Feature spec](../features/import-from-trackers.md)

### Web — email link pages *(later, repo `StreamNext/Web` is empty)*
`/verify-email` and `/reset-password` pages that the auth emails link to (format fixed in [auth.md](../features/auth.md)). Next.js on Vercel; decide CORS vs server-side calls when it starts.

### Later
Movies, the Vercel website, the iOS app, the Android app.
