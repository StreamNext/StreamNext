# StreamNext — Overview

## What it is
<!-- One or two sentences: what the product is, who it's for, the core idea/value proposition. -->

## Goals
<!-- Bulleted list of what the product is trying to achieve. -->
-

## What it is not
<!-- Explicit non-goals — as useful as the goals list for keeping scope honest. -->
-

## Current status
<!-- What's actually live today: which repos exist, which clients are real vs planned,
     where it's deployed (if anywhere yet). Update this the moment any of that changes —
     a stale "current status" is worse than an honest "nothing is live yet". -->
Nothing is implemented yet. `Backend` and `Proto` exist as empty
repo shells (just `.gitignore`/`README.md`). `iOS` and `Android`
haven't been created yet.

## Repos

| Folder | Repo | Purpose |
|---|---|---|
| `StreamNext` (this repo) | StreamNext/StreamNext | Workspace root — onboarding, specs, Claude context |
| `Backend` | StreamNext/Backend | C#/.NET REST + JWT API server |
| `Proto` | StreamNext/Proto | OpenAPI (REST contract) + protobuf (shared client types, if needed) |
| `iOS` | StreamNext/iOS | SwiftUI native iOS app *(not created yet)* |
| `Android` | StreamNext/Android | Kotlin/Compose native Android app *(not created yet)* |
