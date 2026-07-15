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

**My position:** I chose to keep `public=True` as the default.

**Reasoning:** CineLog is a community platform, and a public watchlist lets other users discover films someone plans to watch, not just films they've already seen (which is what the existing collection feature covers). That visibility can spark recommendations or conversations before someone even watches a film — arguably more valuable to the community than sharing what's already been watched. I'm optimizing for the social feature actually working from day one: if watchlists were private by default, most users would never think to change the setting, and the discovery/interaction value of the feature would go largely unused. Public-by-default means the community aspect works without requiring any extra configuration from the user.

**Tradeoff acknowledged:** The real cost is that someone could unintentionally share their viewing plans before they're ready to. For example, a user might be planning a movie night, preparing a surprise for someone, or simply prefer to keep their upcoming watches private for a while. I think this tradeoff is acceptable because it's mitigated, not ignored: adding an explicit `public` parameter to `add_to_watchlist()` (see stretch feature) gives users control to opt out of the public default on a per-entry basis when privacy matters, while keeping the default optimized for the platform's social experience.

## Comment 5 — Sort order

**My position:** I agree with changing the default sort order to date added.

**Reasoning:** I think the maintainer's reasoning is convincing because a watchlist is more like a queue of movies someone plans to watch than a permanent reference list. Showing the most recently added films first makes it easier for users to see their latest additions and keeps the behavior consistent with `get_collection()`, which already uses newest-first ordering.

**Engagement with reviewer's point:** I do think alphabetical ordering has value, particularly for large watchlists where users are looking for a specific title. However, I believe recency better matches the primary workflow of adding and managing films. If both use cases become important, allowing users to choose their preferred sort order in the future would provide the flexibility to support both.

## Comment 6 — Rebase

**What conflicted:** An `add/add` conflict in `.gitignore` — both `main` and my branch independently added the file with slightly different contents (`main`'s version included `.pytest_cache/`, mine didn't).

**How I resolved it:** Kept both sets of entries, since `.gitignore` patterns are additive and combining them is harmless — there was no reason to drop `.pytest_cache/` just because my branch hadn't added it yet.

**How I verified no conflict remains:** After completing the rebase, `git status` showed a clean working tree with no unmerged paths. Running the full test suite afterward surfaced a deeper, pre-existing bug unrelated to the `.gitignore` conflict itself: `WatchlistEntry` was imported and used throughout `watchlist_service.py`, but had never actually been defined in `models.py` — the commit that claimed to add "watchlist model and endpoint" only touched the service and route files. I added the missing `WatchlistEntry` model, mirroring `CollectionEntry`'s structure and incorporating the `public` field (Comment 4) and `date_added`-based sorting (Comment 5). I also fixed two stale docstrings that still described `film_id` as an integer, left over from before the UUID refactor. After these fixes, all 5 tests passed (`pytest tests/ -v`).

## PR Description

<!-- Written at the end — feature overview, design decisions, manual testing steps -->
## PR Description

### What this PR does

Adds a watchlist feature to CineLog, allowing users to save films they want to watch (as distinct from the existing collection feature, which tracks films already watched). Includes:

- A new `WatchlistEntry` model (`models.py`), storing `user_id`, `film_id`, `date_added`, and a `public` visibility flag.
- `add_to_watchlist()` and `get_watchlist()` service functions (`services/watchlist_service.py`), with duplicate-entry prevention (`AlreadyInWatchlistError`) and nonexistent-film handling (`FilmNotFoundError`).
- A `GET /watchlist/<user_id>` endpoint to view a user's watchlist, and a `POST /watchlist/<user_id>/add` endpoint to add a film (`routes/watchlist/watchlist.py`).
- A test covering the nonexistent-film-id case (`tests/test_watchlist.py`).

### Design decisions

**Default visibility (Comment 4):** New watchlist entries default to `public=True`. Reasoning: a watchlist enables film discovery among users before they've watched something, which is more socially useful than the collection feature's "already watched" list, and a public default means this discovery value works without requiring manual configuration from every user. The tradeoff — unintentionally exposing viewing plans someone wanted to keep private — is mitigated by allowing an explicit `public` parameter to opt out per entry.

**Sort order (Comment 5):** `get_watchlist()` returns entries ordered by `date_added` (newest first), matching the maintainer's suggestion and `get_collection()`'s existing convention. A watchlist functions more like an actively-managed queue than a static reference list, so recency is more useful than alphabetical order for the primary use cases (confirming a recent add, deciding what to watch next).

### Manual testing

1. Start the app (e.g. `flask run`, or however this project is normally started).
2. Create a user and a film via the existing collection/film endpoints (or seed the DB directly).
3. Add a film to the watchlist:

Confirm a 201 response and that the returned entry has `public: true` and a `date_added` timestamp.
4. Repeat the same request with the same `user_id`/`film_id` — confirm it fails with `AlreadyInWatchlistError` rather than creating a duplicate.
5. Repeat with a fake/nonexistent `film_id` (e.g. a random UUID) — confirm it fails with `FilmNotFoundError`.
6. View the watchlist:
Confirm films are returned newest-`date_added`-first.
7. Run the automated suite: `pytest tests/ -v` — all 5 tests should pass.
