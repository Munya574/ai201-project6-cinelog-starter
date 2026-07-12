# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude (AI assistant) throughout this project for:
- **Codebase orientation**: Summarized `models.py`, `services/collection_service.py`, and `tests/test_collection.py` to understand naming conventions, deduplication patterns, and test structure before making changes.
- **Understanding patterns**: Asked the AI to walk through what `add_to_collection()`'s duplicate check does so I could write an equivalent for `add_to_watchlist()`.
- **Git mechanics**: Used AI to understand and safely execute the rebase onto UUID main, including resolving conflicts in `models.py` where WatchlistEntry's film_id needed to change from Integer to String(36).
- **Design reasoning**: For Comments 4 and 5 (visibility and sort order), I wrote my own positions and reasoning. I then used AI as a devil's advocate to stress-test my arguments before finalizing them in the pr-response.md.

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
**My position:** Watchlist entries should default to private, not public.

**Reasoning:** Privacy should be the default. A watchlist is inherently personal — it tracks what you're curious about, what's on your mind, what you haven't gotten to yet. Sometimes you don't want others to know what you're watching, especially before you've actually seen it. Making privacy the default respects user autonomy: they can explicitly choose to share if they want to, but they shouldn't have to opt out of exposure. Public sharing should be an intentional, deliberate choice, not something that happens by accident.

**Tradeoff acknowledged:** The obvious cost is a quieter community — fewer public watchlists visible to discover what others are watching. This undermines CineLog's goal as a community app. Some profiles will look empty because users never opt in to sharing. However, I believe privacy-first builds trust, and trust is what gets users to share willingly rather than reluctantly.

## Comment 5 — Sort order
**My position:** The watchlist should default to newest-added first (most recently added first), matching the collection's sort order.

**Reasoning:** Date-added (newest first) gives a sense of freshness and recency. When I open my watchlist, I want to see what I just added — that recent film I'm excited about — near the top. It also helps users track their own behavior: "Did I actually watch that film I added six months ago?" By seeing recent additions first, users can gauge their watchlist habits over time. Additionally, sorting by date-added provides consistency with `get_collection()`, which already sorts newest-first — users expect the same chronological ordering across similar features.

**Engagement with reviewer's point:** I understand the value of alphabetical sorting for discoverability in a long list, so I've implemented date-added as the default with an optional `?sort=title` query parameter. This keeps the default behavior consistent with the collection and optimized for the common case (browsing recent additions), while preserving alphabetical lookup for users who need it when their watchlist grows large.

## Comment 6 — Rebase
**What conflicted:**
While the PR was open, main was refactored to migrate Film IDs from integer to UUID (commit: "refactor: migrate film IDs from integer to UUID"). When rebasing feature/watchlist onto the updated main, two files conflicted:

1. **`.gitignore`** — an add/add conflict. Both branches added a .gitignore. Main's version didn't include `.pytest_cache/`, so I took the union of both.
2. **`models.py`** — the WatchlistEntry class (added by our branch) had `film_id = db.Column(db.Integer, ...)`, but main's refactor changed Film.id to UUID (String(36)). This mismatch would have caused foreign key failures.

**How I resolved it:**
1. Resolved `.gitignore` by keeping all entries from both versions (including `.pytest_cache/`, `.venv/`, etc.).
2. Resolved `models.py` by updating WatchlistEntry's film_id from `db.Column(db.Integer, db.ForeignKey("film.id"))` to `db.Column(db.String(36), db.ForeignKey("film.id"))` to match the UUID migration. This ensures WatchlistEntry can reference the new UUID-based Film IDs.
3. Ran `git add models.py` and `git rebase --continue` to complete the rebase.

**How I verified no conflict remains:**
- `git log --oneline` shows a linear history with no merge commits (the branch was rebased, not merged).
- `git merge-base --is-ancestor origin/main HEAD` confirms the branch sits cleanly on top of main.
- `pytest tests/ -v` passes all 8 tests (4 collection + 4 watchlist) against the UUID codebase, confirming the watchlist code works end-to-end after the migration.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog — a per-user list of films a user wants to watch later, parallel to the existing "collection" (films already watched). It provides:
- `POST /watchlist/<user_id>/add` with body `{ "film_id": "<uuid>" }` — adds a film to the user's watchlist. Returns **201** with the new entry, **404** if the film doesn't exist, **409** if the film is already on that user's watchlist (deduplication), and **400** if `film_id` is missing.
- `GET /watchlist/<user_id>` — returns the user's watchlist. Sorted by date-added (newest first) by default; pass `?sort=title` for alphabetical order.

It follows the conventions already established by the collection service: the `verb_to_noun` naming (`add_to_watchlist`), a custom `AlreadyInWatchlistError` for duplicates, and the same fixture/assertion style in the tests. It also fixes a latent bug the new tests surfaced — `Film` was missing its relationship to `WatchlistEntry`.

### Design decisions
1. **Default visibility = private.** New watchlist entries default to private rather than public. This optimizes for user privacy and trust: sharing should be an explicit, opt-in choice, since content that becomes public is effectively impossible to fully retract. The tradeoff is a quieter community — some profiles will look empty because users never opt in to sharing. However, privacy-first builds trust, and trust is what gets users to share willingly rather than reluctantly. (See Comment 4 for full reasoning.)

2. **Default sort = date-added (newest first), with an opt-in `?sort=title`.** The default matches the collection endpoint (consistency) and treats the watchlist as a "what's fresh to watch next" feed. Alphabetical ordering is preserved as a caller-specified option for finding a specific title in a long list, rather than being discarded. (See Comment 5 for full reasoning.)

### How to manually test
Run everything from the project root so the app and seed snippet share the same database.

```bash
# 1. Start the app (Terminal 1) — runs at http://127.0.0.1:5000
python app.py

# 2. Seed one user + one film and print their IDs (Terminal 2)
python - <<'PY'
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username="demo", email="demo@example.com")
    f = Film(title="Paddington 2", year=2017, genre="Comedy")
    db.session.add_all([u, f]); db.session.commit()
    print("USER_ID:", u.id)
    print("FILM_ID:", f.id)
PY

# 3. Add the film to the watchlist  -> expect HTTP 201
curl -i -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
  -H "Content-Type: application/json" -d '{"film_id": "<FILM_ID>"}'

# 4. Add the SAME film again        -> expect HTTP 409 (deduplication)
curl -i -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
  -H "Content-Type: application/json" -d '{"film_id": "<FILM_ID>"}'

# 5. Add a film that doesn't exist  -> expect HTTP 404
curl -i -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
  -H "Content-Type: application/json" -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'

# 6. View the watchlist (default: newest-added first)
curl http://127.0.0.1:5000/watchlist/<USER_ID>

# 7. View it alphabetically by title
curl "http://127.0.0.1:5000/watchlist/<USER_ID>?sort=title"
```

You can also run the automated suite: `pytest tests/ -v` (8 tests, all passing).

### Commit history
`git log --oneline` on `feature/watchlist` — 8 commits rebased on updated main, no merge commits:

![git log --oneline](docs/Screenshot%202026-07-12%20185942.png)
