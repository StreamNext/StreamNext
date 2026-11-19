# Functional Requirements

## Implemented

### M0 — Foundation
Go backend skeleton with `GET /v1/health`, sops secrets, goose migrations, 90% coverage gate and a Bruno suite; Proto contract with buf module. Implemented locally; CI and deploy are unverified.
→ [Feature spec](features/foundation.md)

### M1 — Auth
Email and password, Sign in with Apple, Google sign-in; 15-minute access JWTs with rotating, device-bound refresh tokens and theft detection; email verification, password reset, change-password, account deletion; push-token registration. Implemented in three slices (Backend 0.2.0 to 0.4.0). Not yet exercised with real Apple/Google tokens or a real mail provider.
→ [Feature spec](features/auth.md)

### M2 — Catalog
Search, series detail and episodes from TheTVDB in English or Spanish per the user's profile language. A series is fetched when first opened and stored in Postgres; the API never refreshes it. A separate `tvdb-sync` job (`make tvdb-sync`, hourly once deployed) applies TheTVDB's updates feed to stored series. Verified against the real API.
→ [Feature spec](features/catalog.md)

<!-- Each further entry, once a feature ships, should look like:

### Feature name
One or two sentences describing what's implemented and its key business rule.
→ [Feature spec](features/feature-name.md)
-->

## Planned

Backend milestones, in build order. Each gets a `docs/features/*.md` spec via `/spec` right before it is implemented.

### M3 — Tracking
Follow and unfollow shows with a list state (watching, planned, paused, dropped). Mark an episode, a season, or everything up to an episode as watched, and unwatch. "Up next" per show and across all shows, plus an upcoming-episodes calendar.

### M4 — Notifications
Builds on M2's `tvdb-sync` job: compare each refreshed series with its previous state to raise **renewed** (a new season, or Ended→Continuing) and **ended** (Continuing→Ended) events. TheTVDB has no separate "cancelled" status. A 15-minute job sends airing-soon alerts, using the series' country to turn air times into UTC. Push goes through FCM on both platforms, only to devices with a live refresh token, with an in-app inbox and per-user and per-show preferences.

### M5 — Episode threads
Posts with one level of replies, reactions, edit and delete, and reports (a post is auto-hidden after N reports). GIFs come from GIPHY. A thread is readable and postable only after the user has watched the episode or unlocked spoilers for that one episode; the unlock is remembered.

### M6 — Analytics
Total shows, episodes and minutes watched; completed shows; a monthly trend; top genres and networks.

### M7 — Discovery
A separate feature with its own weekly job, logic and tables; the catalog is not mirrored for it.
- **v1, the same for everyone:** ranked lists per genre, favoring continuing, upcoming and recently aired shows (people mostly watch recent things).
- **Seeding:** TheTVDB's `/series/filter`, sorted by score. It requires an origin country and language, so candidates are seeded per market (for example US/UK English, Spanish-speaking countries, Korea, Japan); which markets is a product decision for this milestone.
- **Storage:** candidates hold only what a list needs (name, image, year, genres, score, in English and Spanish). Opening one goes through the catalog's fetch-on-miss.
- **Later, if usage grows:** personalized "for you" and "similar to X" from follows and watches, and "people who follow X also follow Y" from our users. TMDB's similar/recommendation endpoints remain an option if TheTVDB's coarse genres limit quality (second API and license).

### M8 — Import from other trackers *(placeholder)*
Import follows and watch history from TV Time and Trakt, mapped to TheTVDB ids. → [Feature spec](../features/import-from-trackers.md)

### Web — email link pages *(later, repo `StreamNext/Web` is empty)*
`/verify-email` and `/reset-password` pages that the auth emails link to (format fixed in [auth.md](../features/auth.md)). Next.js on Vercel; decide CORS vs server-side calls when it starts.

### Later
Movies, the Vercel website, the iOS app, the Android app.
