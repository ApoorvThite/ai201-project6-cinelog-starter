# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used Claude Code to explore the codebase (models.py, collection_service.py, test_collection.py)
before reading the review comments, and to verify that grep found all call sites for
save_to_watchlist before committing the rename. Also used it as a sounding board for Comments 4
and 5 — I wrote draft positions first, then asked for counterarguments. The counterargument on
Comment 4 (opt-in privacy is safer) was one I'd already acknowledged in my draft. The
counterargument on Comment 5 (alphabetical is better for large lists) was real but I still think
recency wins for the reasons below. No AI-generated prose made it into the final responses.

---

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py`. Used grep to find every call site before touching anything —
the only other reference was in `routes/watchlist/watchlist.py` (the import and the one call on
line 32). Updated both. Ran grep again after to confirm zero remaining references.

**How I verified:** `grep -r "save_to_watchlist" --include="*.py" .` returned nothing.
Full test suite still passed (6/6 at that point).

---

## Comment 2 — Deduplication
**What I did:** Added `AlreadyInWatchlistError` (mirroring `AlreadyInCollectionError` in
`collection_service.py`) and a `WatchlistEntry.query.filter_by()` check before inserting, in
exactly the same structure as `add_to_collection()`. If an entry already exists, the error is
raised and no insert is attempted.

**How I verified:** `test_add_to_watchlist_duplicate_raises` confirms the error is raised and
that only one row exists after two identical calls. Full suite passed.

---

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` following the same fixture pattern as
`test_collection.py` (in-memory SQLite app fixture, separate `sample_user` and `sample_film`
fixtures). Wrote `test_add_to_watchlist_nonexistent_film_raises` as the direct equivalent of
`test_add_to_collection_nonexistent_film_raises`. Also added a duplicate test (Comment 2 above
needed one anyway) and a sort order test (Comment 5).

**How I verified:** `pytest tests/test_watchlist.py -v` — 2 passed initially, 3 after sort
order change. `pytest tests/ -v` — 7/7.

---

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default.

**Reasoning:** CineLog is described as a *community* film tracking app. The watchlist is the
social signal — "here's what I'm planning to watch" — and sharing that is the primary reason to
have a public-facing watchlist at all. Making it opt-in (public=False default) puts friction
on the exact behavior the feature is designed to support.

There's also a consistency argument within the codebase: `CollectionEntry` has no visibility
field at all. Every logged film is effectively public. If the project treated privacy as the
default posture, the collection would have a visibility toggle too. The absence of one signals
that the project assumes openness. Defaulting the watchlist to public=True is consistent with
that posture.

Operationally, if a user doesn't want a public watchlist they can set `public=False` — but if
the default were False, most users who don't read the docs would never turn on sharing, and the
social feature would go unused.

**Tradeoff acknowledged:** Opt-out privacy is legitimately worse than opt-in from a user
expectation standpoint. A new user adding a film to their watchlist may not realize it's
immediately visible to others. If CineLog ever wants to be privacy-safe by default (GDPR
territory, or simply respecting user surprise), `public=False` with an explicit sharing step is
the right call. I'd revisit this if the app adds user-facing account settings or a privacy
preferences page — at that point the default matters less because users have a clear way to
change it.

---

## Comment 5 — Sort order
**My position:** Changed to `date_added` descending (newest first). I agree with the
reviewer's preference here.

**Reasoning:** The reviewer's consistency argument is the strongest one. Both `get_collection()`
and `get_watchlist()` return lists of films for a user. If one sorts newest-first and the other
sorts alphabetically, a caller using both endpoints has to hold two different mental models of
what "order" means. That's unnecessary cognitive load for a very thin benefit.

Recency also makes more sense semantically for a watchlist. When I add a film to my watchlist,
it's usually because I just heard about it or someone recommended it — it's top of mind. Showing
it first lets me act on it. A film I added two years ago and still haven't watched is lower
priority by revealed preference. Alphabetical order doesn't reflect any of that.

**Engagement with reviewer's point:** The reviewer frames this as a consistency issue, and I
think that's exactly right. The API surface shouldn't require callers to know that collection
sorts by time while watchlist sorts by name. The only case where alphabetical wins is a large
watchlist where you're scanning for a specific title — but that's what a search/filter endpoint
is for, not a list sort. Implemented: changed `.join(Film).order_by(Film.title.asc())` to
`.order_by(WatchlistEntry.date_added.desc())`.

---

## Comment 6 — Rebase
**What conflicted:** `models.py`. The main branch migrated `Film.id` from `db.Integer` to
`db.String(36)` (UUID), and updated `CollectionEntry.film_id` to match. The feature/watchlist
branch added `WatchlistEntry` using the old `db.Integer` for `film_id`. When rebasing,
`models.py` had a conflict in the `WatchlistEntry` class definition — the incoming main version
didn't include `WatchlistEntry` at all (it was added by this branch), but the `Film.id` type
change meant `WatchlistEntry.film_id` needed updating too.

Also updated the docstring in `watchlist_service.py` that still said `film_id (int)` and the
route comment that said `"film_id": <int>`.

**How I resolved it:** Updated `WatchlistEntry.film_id` from `db.Column(db.Integer, ...)` to
`db.Column(db.String(36), ...)` to match the migrated `Film.id` type. Updated the `film_id`
ForeignKey reference accordingly. Updated stale docstrings to say UUID instead of int.

**How I verified no conflict remains:** `git log --oneline` shows no merge commits.
`python -m pytest tests/ -v` — all tests pass. `git status` is clean.

---

## PR Description

### Watchlist feature

Adds a watchlist (films a user plans to watch) alongside the existing collection (films already
watched). Users can add films to their watchlist, retrieve it sorted by most-recently-added, and
the feature enforces deduplication so the same film can't be added twice.

### Endpoints

- `GET /watchlist/<user_id>` — returns the user's watchlist, newest first
- `POST /watchlist/<user_id>/add` — adds a film; body: `{ "film_id": "<uuid>" }`

### Design decisions

- **Default visibility (`public=True`):** Watchlist entries are public by default. CineLog is a
  community app, and the collection has no visibility field at all (implicitly public). Defaulting
  to open is consistent with the project's social posture. Users who want privacy set
  `public=False` explicitly.
- **Sort order (`date_added` desc):** Changed from alphabetical to newest-first to match
  `get_collection()`. Consistency across both list endpoints reduces API cognitive load; recency
  is also more semantically meaningful for a "to watch" queue.

### Manual testing

```bash
# Start the app
python app.py

# Add a film (replace UUIDs with real ones from your db)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'

# View watchlist
curl http://127.0.0.1:5000/watchlist/<user_id>

# Duplicate should return 500 (AlreadyInWatchlistError — wire up error handler to get 409)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<same_film_uuid>"}'
```

### Tests

```bash
pytest tests/ -v   # 7 passed
```
