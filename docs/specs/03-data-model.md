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
| `oauth_account` | M1 | `(provider, subject)` unique; `user_id` |
| `email_token` | M1 | `token_hash`; `user_id`; `purpose` (verify / reset); `expires_at`; `used_at` |
| `device` | M1 | `(user_id, device_id)` unique; `fcm_token`; `platform` (ios / android) |
| `series` | M2 | `id` = TheTVDB id (bigint); `name`; `slug`; `overview`; `image`; `status` (continuing / ended / upcoming); `first_aired`, `last_aired`, `next_aired`; `airs_time`; `country`; `network`; `average_runtime`; `score`; `synced_at` |
| `genre`, `series_genre` | M2 | `genre.id` = TheTVDB id |
| `season` | M2 | `id` = TheTVDB id; `series_id`; `number`; default season type only |
| `episode` | M2 | `id` = TheTVDB id; `series_id`; `season_number`; `number`; `name`; `overview`; `aired_date`; `runtime`; `image` |
| `follow` | M3 | PK `(user_id, series_id)`; `state` (watching / planned / paused / dropped); `notify` bool; `created_at` |
| `episode_watch` | M3 | PK `(user_id, episode_id)`; `watched_at` is set by the client or an import, not `now()`, so imported history keeps its real dates; no rewatches in v1 |
| `notification` | M4 | `id`; `user_id`; `kind`; `series_id`; `episode_id`; `created_at`; `sent_at`; `read_at`; a unique dedupe key so a job re-run never notifies twice |
| `sync_state` | M4 | `key` PK, `value` — holds the TheTVDB `/updates` cursor |
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
