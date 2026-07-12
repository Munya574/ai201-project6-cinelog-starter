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

**How I verified:**

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
