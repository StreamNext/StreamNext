# Data Model

> **Status: first cut, not yet migrated.** Each table below is created by the milestone noted next to it. Each milestone's migration and its feature spec should make columns more precise. Update this file in the same change, and move the table into "Entity reference" once its migration exists.

## Logical model

```
User
  ├── OAuthAccount            (apple | google)
  ├── RefreshToken            (per device)
  ├── EmailToken              (verify | reset)
  ├── Device                  (FCM token)
  ├── Follow ──▶ Series
  ├── EpisodeWatch ──▶ Episode
  ├── SpoilerUnlock ──▶ Episode
  ├── Notification ──▶ Series / Episode
  └── Post ──▶ Episode
        ├── Post (reply, 1 level)
        ├── PostReaction
        └── PostReport

Series (TheTVDB cache)
  ├── SeriesGenre ──▶ Genre
  ├── Season
  └── Episode
SyncState (TheTVDB /updates cursor)
```

## Planned tables

| Table | Milestone | Key columns / rules |
|---|---|---|
| `follow` | M3 | PK `(user_id, series_id)`; `state` (watching / planned / paused / dropped); `notify` bool; `created_at` |
| `episode_watch` | M3 | PK `(user_id, episode_id)`; `watched_at` is set by the client or an import, not `now()`, so imported history keeps its real dates; no rewatches in v1 |
| `notification` | M4 | `id`; `user_id`; `kind`; `series_id`; `episode_id`; `created_at`; `sent_at`; `read_at`; a unique dedupe key so a job re-run never notifies twice |
| `post` | M5 | `id`; `episode_id`; `user_id`; `parent_id` (null = top level; replies are one level deep); `body`; `giphy_id`; `created_at`, `edited_at`, `deleted_at`; `hidden` |
| `post_reaction` | M5 | PK `(post_id, user_id, kind)` |
| `post_report` | M5 | PK `(post_id, reporter_id)`; `reason` |
| `spoiler_unlock` | M5 | PK `(user_id, episode_id)`; `created_at` |

## Entity reference

<!-- Move a table here, with its full column list, once its migration exists. -->

### `users` (uuid PK, migration 2)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK, `gen_random_uuid()` |
| `email` | citext | unique, case-insensitive; stored lowercased and trimmed |
| `password_hash` | text | null | argon2id PHC string; null for social-only accounts (slice 3) |
| `email_verified_at` | timestamptz | null until verified (slice 2) |
| `display_name` | text | 1–40 characters |
| `timezone` | text | IANA name, default `UTC` |
| `created_at`, `updated_at` | timestamptz | |

### `refresh_token` (uuid PK, migration 2)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `user_id` | uuid | FK → `users`, `ON DELETE CASCADE` |
| `device_id` | uuid | client-generated; binds the token to a device |
| `token_hash` | text | unique; SHA-256 hex of the opaque token (the token itself is never stored) |
| `lifetime_seconds` | integer | carried forward on rotation so the window slides |
| `expires_at` | timestamptz | |
| `created_at` | timestamptz | |
| `revoked_at` | timestamptz | null while live |
| `replaced_by` | uuid | FK → `refresh_token`, `ON DELETE SET NULL`; the successor after a rotation (reuse and grace logic) |

Indexes: partial unique `(user_id, device_id) WHERE revoked_at IS NULL` (one live token per device), `user_id`.

### `email_token` (uuid PK, migration 3)
Single-use tokens in verification and password-reset emails.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `user_id` | uuid | FK → `users`, `ON DELETE CASCADE` |
| `purpose` | text | `verify` or `reset` (CHECK); a token only works for its own purpose |
| `token_hash` | text | unique; SHA-256 hex (the token only ever exists in the email) |
| `expires_at` | timestamptz | 24h for `verify`, 1h for `reset` |
| `created_at` | timestamptz | also drives the 60-second per-address email cooldown |
| `consumed_at` | timestamptz | set when used, or when a newer token of the same purpose supersedes it |

Index: `(user_id, purpose, created_at DESC)`.

### Catalog tables (migration 5)
TheTVDB's data for the series someone has opened. Ids are TheTVDB's own (`bigint`), so sync and later features need no mapping table. Everything under `series` cascades from it.

| Table | Key columns / rules |
|---|---|
| `series` | `id`; `slug`; `original_name`, `original_overview`, `original_language`/`original_country` (TheTVDB's three-letter codes); `image`; `status` (`continuing`, `ended`, `upcoming`, `unknown`); `first_aired`, `last_aired`, `next_aired` (dates); `airs_time` (local to the country), `airs_days` (text[]); `network`; `average_runtime` (0 = unknown); `tvdb_updated_at`; `synced_at` (indexed, drives the 30-day safety net) |
| `series_translation` | PK `(series_id, language)`, `language` in `en`, `es`; `name`, `overview` |
| `genre` | `id`, English `name`, `slug`; shared by series. Spanish names are mapped in code. |
| `series_genre` | PK `(series_id, genre_id)` |
| `season` | `id`; `(series_id, number)` unique; `name`, `image`; default season order only; number 0 = specials |
| `episode` | `id`; `series_id`, `season_number`, `number`, `absolute_number`; `original_name`, `original_overview`; `aired` (date, null while unannounced); `runtime`; `image`; `finale_type`. **Upserted by id** on every refresh and only vanished episodes are deleted, so later references survive. Index `(series_id, season_number, number)` and a partial index on `aired`. |
| `episode_translation` | PK `(episode_id, language)`; `name`, `overview` |
| `sync_state` | `key` PK, `value`, `updated_at`; holds the `tvdb-sync` cursor |

`users` also gained `language` (`en` default, `es`; CHECK).

### `oauth_account` (uuid PK, migration 4)
A Sign in with Apple / Google identity linked to a user.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `user_id` | uuid | FK → `users`, `ON DELETE CASCADE` |
| `provider` | text | `apple` or `google` (CHECK) |
| `subject` | text | the provider's stable id for the person |
| `email` | text | null; what the provider reported when the identity was linked (informational) |
| `created_at` | timestamptz | |

Unique `(provider, subject)`: an identity belongs to exactly one user. Index: `user_id`.

### `device` (uuid PK, migration 4)
A user's device and its push (FCM) token.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `user_id` | uuid | FK → `users`, `ON DELETE CASCADE` |
| `device_id` | uuid | client-generated; the same id as `refresh_token.device_id` |
| `fcm_token` | text | **unique**: one token, one owner; registering it again moves it |
| `platform` | text | `ios` or `android` (CHECK) |
| `created_at`, `updated_at` | timestamptz | |

Unique `(user_id, device_id)`. For M4: push only to devices that also have a live refresh token for that `(user_id, device_id)`.

## Key types

### ID types
- Our own rows use `uuid` primary keys.
- Catalog rows (`series`, `season`, `episode`, `genre`) use TheTVDB's integer id (`bigint`) as the primary key, so sync and imports join without a mapping table.

## Enums

Postgres `text` columns with `CHECK` constraints, mirrored as enums in the OpenAPI contract:
- `follow.state`: `watching`, `planned`, `paused`, `dropped`
- `series.status`: `continuing`, `ended`, `upcoming`
- `notification.kind`: `airing_soon`, `new_episode`, `new_season`, `renewed`, `ended`
- `device.platform`: `ios`, `android`
- `email_token.purpose`: `verify`, `reset`

## Migration history

| # | File | Change |
|---|---|---|
| 1 | `00001_extensions.sql` | `citext` extension (case-insensitive emails from M1) |
| 2 | `00002_auth.sql` | `users`, `refresh_token` and their indexes |
| 3 | `00003_email_token.sql` | `email_token` (verification and reset tokens) |
| 4 | `00004_social_and_devices.sql` | `oauth_account`, `device` |
| 5 | `00005_catalog.sql` | `series`, translations, genres, seasons, episodes, `sync_state`, `users.language` |
