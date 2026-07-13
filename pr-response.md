# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention already used by `add_to_collection()` in `services/collection_service.py`. Updated the one call site in `routes/watchlist/watchlist.py` (`add_film` route), including its import statement.
**How I verified:** Ran `grep -rn "save_to_watchlist" . --include=*.py` — no remaining references. Confirmed the app still boots (`create_app()` succeeds, all blueprints register) after the rename.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a dedup check inside `add_to_watchlist()`, matching the exact pattern used by `add_to_collection()` in `services/collection_service.py`: after confirming the film exists, query `WatchlistEntry` by `user_id`+`film_id`, and raise if an entry already exists (same behavior as collection — raise, not silent no-op).
**How I verified:** Ran a manual script that adds the same film to a user's watchlist twice — the first call succeeds and returns the new entry, the second raises `AlreadyInWatchlistError` as expected.

## Comment 3 — Missing test
**What I did:**
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
