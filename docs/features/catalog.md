# Feature: Catalog

## Status
planned

## GitHub issue
<!-- Optional -->

## Overview
M2. Search TV series and read a series' details, seasons and episodes. The data comes from TheTVDB v4 through our licensed project key. The backend keeps its own copy in Postgres, as TheTVDB itself recommends. The first request for a series fetches it from TheTVDB and stores it, and later requests read Postgres. Clients never call TheTVDB.

Everything after this builds on the catalog: tracking (M3), sync and notifications (M4), episode threads (M5), analytics (M6) and discovery (M7).

## Business rules
**TheTVDB client** (`internal/tvdb`)
- **Login:** `POST /login {apikey}` with the licensed key (no PIN). The token lasts a month. It is kept in memory and renewed after 25 days, or once immediately when TheTVDB answers 401.
- **Connection:** base URL `https://api4.thetvdb.com/v4` (configurable, so tests and the CI stub can replace it), 10-second timeout.
- **Failures:** a network error, a 5xx or a malformed body is "catalog unavailable". TheTVDB's 404 is "not found".
- **Not configured:** if `TVDB_API_KEY` is not set, the catalog endpoints answer 503 `unavailable`, the same pattern as the social providers.

**Language** (English and Spanish)
- **Where it comes from:** every user has a `language`, `en` (default) or `es`, set with `PATCH /v1/me` and returned by `GET /v1/me`. It lives on the profile, not in a request header, because M4 writes push notifications on the server with no request to read a header from.
- **Fallback:** names and overviews are returned in the user's language, falling back to English, then to the show's original language. The original-language name is always returned as well.
- **Fetching:** TheTVDB language codes are `eng` and `spa`.
  - Series translations come with the extended record (`meta=translations`).
  - Episode translations need one paginated pass per language (`/series/{id}/episodes/default/{lang}`, which returns the episodes with their names in that language, empty where there is no translation), so a full fetch makes three passes: original, English, Spanish. A series costs about 5 requests.
  - Search results already include translations.
- **Genres:** TheTVDB only names genres in English, so Spanish genre names come from a small fixed map in our code, keyed by TheTVDB genre id, with English as the fallback for any genre missing from it.
- **Enums:** status values are enums, and clients translate them.

**Search** (`GET /v1/search/series`)
- **Proxying:** forwards to TheTVDB `/search?type=series`, 20 results per page (`limit`/`offset`; the response says how many results exist, which gives `hasMore`).
- **Input:** `query` is trimmed and must be 2–100 characters; `page` is ≥ 1, at most 25 (TheTVDB caps search at 5,000 results).
- **Results:** summaries only (id, name, year, network, status, poster thumbnail, short overview). They are **not stored**: a series is stored when its detail is first opened.
- **Caching:** identical searches (same query, page and language) are cached for 10 minutes. *shortcut: in memory, per instance, bounded size; move to a shared cache if search traffic grows.*

**Series and episodes** (`GET /v1/series/{seriesId}`, `GET /v1/series/{seriesId}/episodes`)
- **Ids:** `seriesId` is TheTVDB's integer id, used as our primary key too, so sync and imports need no mapping table.
- **Fetch on miss only:** a stored series is always served from Postgres, and the API never refreshes it during a request. A series that isn't stored is fetched (the extended record with `meta=translations` and `short=true`, plus every page of episodes in the series' default order), stored **in one transaction**, then served.
- **Freshness is the sync job's job** (see below), not the API's.
- **Failures:** a stored series never depends on TheTVDB being up. For one that isn't stored, a TheTVDB failure is 503, and an id TheTVDB doesn't know is 404. `syncedAt` tells clients when the copy was last refreshed.
- **What is stored for a series:**
  - Name and overview in English and Spanish where TheTVDB has them, with the original-language name and overview kept alongside.
  - Status: `Continuing`, `Ended` and `Upcoming` map to `continuing`, `ended` and `upcoming`; anything else is `unknown`.
  - First, last and next aired dates; air time and days.
  - Original country and language.
  - Network: the latest network, else the original one.
  - Average runtime, genres, and seasons in the default order (season 0 is specials).
- **What is stored per episode:** season number, number, absolute number, name and overview (original, plus English and Spanish translations where they exist), aired date (null while unannounced), runtime, image, finale type.
- **Air times:** stored as TheTVDB gives them, local to the series' country. Turning them into UTC instants is M4's job.
- **Writing a refresh:** episodes are **upserted by their TheTVDB id**, and only episodes TheTVDB no longer lists are deleted. Ids never change, so anything that later references an episode (watch history, threads) survives a refresh. Translations, genres and seasons are rewritten.
- **Duplicate fetches:** two simultaneous first requests for the same series both fetch it; the second write is identical, so this is allowed. *shortcut: no request coalescing; add it if TheTVDB usage becomes a problem.*
- **Episodes endpoint:** `?season=N` limits it to one season. Without it, all episodes come back in season and episode order.

**Sync job** (`cmd/jobs/tvdb-sync`): keeps every stored series fresh, so the API never has to.
- **What it reads:** TheTVDB's `/updates?since=<cursor>&type=series`, following every page (pages are numbered from 0, 500 records each). *Verified against the real API on 2026-11-19:* an update to an episode or season is always accompanied by an update to its series (40 of 40 sampled episode updates, 15 of 15 newly created episodes), and the series' `lastUpdated` moves with it, so the `series` feed alone covers new, changed and removed episodes. The other feeds can't be used: `episodes` and `seasons` records don't name their series, and `translatedseries` / `translatedepisodes` carry translation-row ids that match no series or episode.
- **What it refreshes:** only series **we have stored**; the rest of the feed is ignored. Each affected series is re-fetched in full through the same code as a first fetch and saved in one transaction.
- **Safety net:** every run also refreshes up to 50 stored series whose last sync is more than 30 days old, to catch anything the feed missed (translation-only changes, for instance).
- **Deletions:** a `delete` record for a stored series removes it (children cascade); a stored series TheTVDB answers 404 for is removed too. *M3 must revisit this once watch history references episodes.*
- **Cursor:** kept in `sync_state` and advanced only after a run with **no failures**, to that run's start time (the next run reads from 5 minutes earlier), so a failed run is repeated. TheTVDB refuses a `since` older than 3 months (it answers 400), so a cursor older than 80 days, or none, means every stored series is re-fetched once.
- **Running it:** locally with `make tvdb-sync ENV=dev`; hourly by Cloud Scheduler once deployment exists (on hold).
- **Concurrent writes:** the job and an API miss can save the same series at once. Both write a full, consistent copy in one transaction, so the last one wins harmlessly.

**Access**
- **Sign-in required:** every catalog endpoint requires a bearer token, so our licensed key can't be used as an open proxy. The global rate limit applies.
- **Images:** TheTVDB artwork URLs are returned as they are, and clients load them from TheTVDB's CDN.

## Clients
- Backend + Proto now; iOS and Android later.
- If TheTVDB's license terms for our key require attribution, the clients must show it (for example "Data provided by TheTVDB"). Confirm the terms.

## API endpoints
| Endpoint | Purpose | Errors |
|---|---|---|
| `GET /v1/search/series?query=&page=` | search, returns `{results: [SeriesSummary], page, hasMore}` | 400, 401, 503 |
| `GET /v1/series/{seriesId}` | series detail: fields above, genres, seasons with episode counts, `syncedAt` | 401, 404, 503 |
| `GET /v1/series/{seriesId}/episodes?season=` | episodes, in order | 400, 401, 404, 503 |
| `GET /v1/me`, `PATCH /v1/me` *(changed)* | profile gains `language` (`en` \| `es`) | as before |

## Data model
Migration `00005_catalog`. Catalog rows use TheTVDB's ids (`bigint`) as primary keys.
- **`users`** *(changed)*: `language` text NOT NULL DEFAULT `'en'`, CHECK `en` or `es`.
- **`series`:**
  - Names: `id`, `slug`, `original_name`, `original_overview`, `image`.
  - Status and dates: `status` (CHECK), `first_aired`, `last_aired`, `next_aired` (dates), `airs_time`, `airs_days` (text[]).
  - Origin and runtime: `original_country`, `original_language`, `network`, `average_runtime`.
  - Bookkeeping: `tvdb_updated_at`, `synced_at`.
- **`genre`:** `id`, `name`, `slug`. **`series_genre`:** `(series_id, genre_id)`.
- **`sync_state`:** `key` PK, `value`, `updated_at`; holds the `/updates` cursor.
- **`season`:** `id`, `series_id`, `number`, `name`, `image`.
- **`series_translation`:** `(series_id, language)` PK, `name`, `overview`.
- **`episode`:** `id`, `series_id`, `season_number`, `number`, `absolute_number`, `original_name`, `original_overview`, `aired` (date), `runtime`, `image`, `finale_type`.
- **`episode_translation`:** `(episode_id, language)` PK, `name`, `overview`. Index `(series_id, season_number, number)`, and `aired` for the calendar later.
- Children cascade from `series`.

## Testing
- **TheTVDB client:** `httptest` with fixtures **recorded once from the real API** with our key (`cmd/tvdbrecord`). They are trimmed to the fields we use, and descriptive text (overviews) is replaced by placeholders, because the repo is public and we must not republish TheTVDB's content.
  - Covers login, token renewal on 401, pagination, translations, 404 vs 5xx, malformed bodies, timeouts.
- **Live check (opt-in):** `make tvdb-live` runs a build-tagged test against the real API with the key from sops, to catch API drift. Not run in CI unless a secret is added later.
- **Service:** in-memory fakes. Covers fetch-on-miss, stored series served with TheTVDB down, 404 vs 503, status mapping, and the language fallback (Spanish, then English, then original), including the genre name map.
- **Repository:** Postgres tests. Covers replace-in-one-transaction, vanished episodes removed, genres, cascade, the sync cursor.
- **Sync job:** fakes. Covers only stored series refreshed, the cursor advancing only after a clean run, the 80-day fallback, deletion, the 30-day safety net, and a rerun after a failure doing no harm.
- **REST:** handlers, sign-in required (contract-driven), `int64` path parameter binding, query validation.
- **Bruno:** the CI compose stack runs a small TheTVDB stub (`cmd/tvdbstub`, serving the same fixtures), so search, detail and episodes run end to end without the real key, in both languages (switching the profile to `es` changes the names returned).
- **Fixtures:** `make tvdb-record` (needs the key) calls the real API for a fixed set of series, trims each response to the fields we read, replaces overviews with placeholders, and writes `internal/tvdb/stub/testdata/`. The same fixtures drive the unit tests and the CI stub. Real series names are kept (short factual titles). The updates feed is time-dependent, so its tests use hand-built responses.

## Config
`TVDB_API_KEY` (sops / Secret Manager; unset means the catalog is disabled with 503), `TVDB_BASE_URL` (optional).

**The key must not be pasted into chat.** Add it to the encrypted dev secrets yourself:
1. `make secrets-decrypt` (from `Backend/`).
2. Add `TVDB_API_KEY=...` to `dev.env`.
3. `make secrets-encrypt`.
4. Delete `dev.env`.

## Decisions
1. **Language:** English and Spanish now, chosen per user on the profile, with the fallback Spanish/English → English → original (chosen 2026-11-19).
2. **Catalog requires sign-in** (chosen 2026-11-19).
3. **Fixtures:** real responses, trimmed, with placeholder text (chosen 2026-11-19).
4. **Fetch on miss + a separate sync job** (option B, chosen 2026-11-19): the API never refreshes in a request; `tvdb-sync` applies `/updates` to stored series. No full mirror of TheTVDB: it costs infrastructure for data nobody asked for, and people mostly watch recent shows.
5. **Images:** TheTVDB URLs passed through (proposed default); proxying and caching our own copies can come later.
6. **Profile language** defaults to `en` and is changed with `PATCH /v1/me` (proposed default); clients set it from the device locale right after sign-up.
7. **Discovery is not part of the catalog** (chosen 2026-11-19): M7 gets its own job, logic and tables. The catalog stores only series someone opened or followed.

## Implementation notes
<!-- Filled in after implementation (Phase 5) -->

## Change log
- 2026-11-19: Initial spec (@mauro)
- 2026-11-19: Decisions: English + Spanish per user, sign-in required, trimmed real fixtures
- 2026-11-19: Option B: fetch on miss, `tvdb-sync` job brought forward from M4; discovery kept separate
- 2026-11-19: Sync reads only the `series` updates feed (verified against the real API), plus a 30-day safety net; episodes are upserted by id
