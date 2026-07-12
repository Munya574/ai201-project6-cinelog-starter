# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the naming convention already used by `add_to_collection()` in the collection service. Updated the one call site in `routes/watchlist/watchlist.py` — both the `import` statement and the function call inside the `add_film` endpoint.

**How I verified:**
Ran a project-wide search for the old name `save_to_watchlist` to confirm no other call sites remained. Ran `pytest tests/ -v` and confirmed all 4 collection tests still pass, verifying that the rename didn't break anything and the watchlist module still imports cleanly.

## Comment 2 — Deduplication
**What I did:**
Followed the exact pattern used by `add_to_collection()` in `services/collection_service.py`:
- Added an `AlreadyInWatchlistError` exception class at the top of `services/watchlist_service.py`, mirroring `AlreadyInCollectionError`.
- In `add_to_watchlist()`, after the film-exists check, I query for an existing `WatchlistEntry` with the same `user_id` and `film_id` using `filter_by(...).first()`. If one exists, I raise `AlreadyInWatchlistError` before creating a new entry.
- Updated the route (`routes/watchlist/watchlist.py`) to catch both `FilmNotFoundError` (returning **404**) and the new `AlreadyInWatchlistError` (returning **409 Conflict**) with proper error handling in a try-except block.

**How I verified:**
The deduplication check mirrors the pattern in `add_to_collection()`, where an existing entry lookup returns `None` on no duplicate and a truthy `WatchlistEntry` when there is one. Ran `pytest tests/ -v`; all 4 collection tests still pass, confirming the added logic and route error-handling didn't break the happy path.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py`, modeled directly on the collection tests. Reused the same three fixtures (`app` with an in-memory SQLite DB, `sample_user`, `sample_film`) and wrote four comprehensive tests:
- `test_add_to_watchlist_creates_entry`: Verifies basic add functionality
- `test_add_to_watchlist_duplicate_raises`: Tests the deduplication logic
- `test_add_to_watchlist_nonexistent_film_raises`: Tests that adding a non-existent film raises `FilmNotFoundError`
- `test_get_watchlist_returns_alphabetical_by_default`: Tests the sort order

While adding the sort-order test, we discovered a latent bug: `get_watchlist()` referenced `entry.film`, but the `Film` model had no relationship defined for `WatchlistEntry`. I fixed this by adding `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to the `Film` class, mirroring the existing `collection_entries` relationship.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` and all 4 watchlist tests pass. Then ran the full suite `pytest tests/ -v`; all 8 tests pass (4 collection + 4 watchlist), confirming the new tests integrate cleanly and the relationship fix allows `get_watchlist()` to load films correctly.

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
