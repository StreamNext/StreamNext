# Feature: Import From Trackers

## Status
planned

## GitHub issue
<!-- Optional: GitHub issue tracking this feature, e.g. #42 or https://github.com/org/repo/issues/42 -->

## Overview
Placeholder. Lets a new user bring their follows and watch history from another tracker, so switching to StreamNext doesn't mean re-marking hundreds of episodes. First candidate sources: TV Time and Trakt. Others can be added as needed.

## Business rules
<!-- To be decided before implementation. Starting points: -->
- Every imported show and episode must map to a TheTVDB id. Anything that doesn't map is reported back to the user and never guessed silently.
- Trakt exposes TheTVDB ids directly. A TV Time export may need `/search/remoteid` or a title + year match.
- An import writes `follow` and `episode_watch` rows, keeping the original `watched_at` so analytics reflect real history.
- An import is idempotent: running it twice creates no duplicates and never overwrites watches the user already has.
- Imports run asynchronously as a job, ending with a report (imported / skipped / unmatched).
- Open questions: TV Time data-export format, and whether Trakt is imported via OAuth API or JSON export.

## Clients
<!-- TBD: likely iOS + Android (file picker / OAuth), maybe web. -->
-

## API endpoints
<!-- TBD — e.g. start an import, poll its status, fetch its report. -->
-

## Implementation notes
<!-- Filled in after implementation (Phase 5) -->

## Change log
- 2026-11-19: Initial placeholder spec (@mauro)
