# StreamNext — Overview

## What it is
StreamNext is a TV show tracker. Users follow shows and mark the episodes they've watched. They get push notifications when an episode is about to air, when a show is renewed, and when it ends. Each episode has a spoiler-gated discussion thread. The app also shows viewing analytics and recommends similar shows. Movies are an optional later addition. Show data comes from TheTVDB v4.

## Goals
- Track followed shows and watched episodes, and show what's up next for each show.
- Notify users before a followed show airs, and when it is renewed or ends.
- One discussion thread per episode (opinions, GIF memes via GIPHY), hidden until you've watched that episode or explicitly accept spoilers for it.
- Personal analytics: shows, episodes, minutes watched, trends, favourite genres and networks.
- Discovery: similar shows and "for you" picks based on what you watch.
- Import history from other trackers such as TV Time and Trakt (planned).

## What it is not
- Not a streaming player or a link aggregator: we never point to where a show can be watched.
- Not a data source of its own: show, season and episode facts come from TheTVDB and are cached, never edited by users.
- No user image uploads in v1: memes are GIPHY GIFs only.

## Current status
M0 (foundation), M1 (auth) and M2 (catalog) are implemented: `Backend` 0.5.0 and `Proto` 0.5.0 are on `develop` on GitHub with CI green, and nothing is deployed. `iOS` and `Android` haven't been created. The backend is being built first (see the milestones in `01-requirements.md`).

## Repos

| Folder | Repo | Purpose |
|---|---|---|
| `StreamNext` (this repo) | StreamNext/StreamNext | Workspace root — onboarding, specs, Claude context |
| `Backend` | StreamNext/Backend | Go REST + JWT API server, plus scheduled jobs |
| `Proto` | StreamNext/Proto | OpenAPI (REST contract) + protobuf shared message types, published to BSR `buf.build/streamnext` |
| `iOS` | StreamNext/iOS | SwiftUI native iOS app *(not created yet)* |
| `Android` | StreamNext/Android | Kotlin/Compose native Android app *(not created yet)* |
| — | Vercel | Website: marketing and account pages *(later)* |
