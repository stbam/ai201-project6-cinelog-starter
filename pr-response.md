# PR Response Doc — CineLog Watchlist Feature

## AI Usage

<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename

**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention (documented in CONTRIBUTING.md and used consistently in `add_to_collection()`, `remove_from_collection()`, `get_collection()`). Updated the one call site in `routes/watchlist/watchlist.py` (both the import statement and the function call in `add_film()`).
**How I verified:** Ran `grep -rn "save_to_watchlist" .` across the project to confirm no references to the old name remained. Ran `pytest tests/ -v` — all 4 existing tests still passed.

## Comment 2 — Deduplication

**What I did:** Added a deduplication check to `add_to_watchlist()` in `services/watchlist_service.py`, following the same pattern used in `add_to_collection()`. Before creating a new `WatchlistEntry`, the function now queries for an existing entry with the same `user_id` and `film_id` using `WatchlistEntry.query.filter_by(...).first()`. If one is found, it raises a new `AlreadyInWatchlistError` exception (mirroring `AlreadyInCollectionError`) instead of silently creating a duplicate.
**How I verified:** Ran `pytest tests/ -v` to confirm all existing tests still pass after the change. (I'll add a dedicated test for this behavior as part of the stretch goals / Comment 3 work.)

## Comment 3 — Missing test

**What I did:** Created `tests/test_watchlist.py` with a test `test_add_to_watchlist_nonexistent_film_raises`, following the exact fixture and assertion pattern from `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`. Copied the `app` and `sample_user` fixtures (test files don't share fixtures automatically without a `conftest.py`, so each test file defines its own). The test calls `add_to_watchlist()` with a fake UUID and asserts it raises `FilmNotFoundError`, same as the collection equivalent.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — passed. Then ran the full suite with `pytest tests/ -v` to confirm all 5 tests (4 existing + 1 new) pass together.

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
