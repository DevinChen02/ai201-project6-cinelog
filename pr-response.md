# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->
- **Orientation (Milestone 1):** Used AI to summarize `models.py` and `collection_service.py` and to walk through `add_to_collection()` before reproducing its dedup pattern — then verified every summary against the actual code.
- **Devil's advocate on Comments 4 & 5:** Ran my draft design-decision responses through an AI as a hostile reviewer ("what counterargument would a careful reviewer raise; what tradeoff am I not acknowledging?"). For Comment 4 it pushed back that "low sensitivity" is my assumption and that privacy-by-default is the legally/industrially safer norm — so I expanded the Tradeoff section to name that explicitly and added the telemetry-based revisit condition. For Comment 5 it noted that pure deference to the maintainer is weak, which prompted me to add the independent `get_collection()` consistency argument. Final wording and positions are my own.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to follow the project's `verb_to_noun` naming convention (matching `add_to_collection()`). Updated the one call site in `routes/watchlist/watchlist.py` — both the import on line 8 and the call on line 32.
**How I verified:** Ran a project-wide search for `save_to_watchlist` before renaming, which surfaced exactly 3 references across 2 files (the definition, the import, and the call). After editing, I re-ran the same search and it returned zero matches, confirming no call site was missed. The full suite (`pytest tests/ -v`) still passes.

## Comment 2 — Deduplication
**What I did:** Added a duplicate guard to `add_to_watchlist()` following the same pattern as `add_to_collection()` in `services/collection_service.py`. After the film-existence check and before creating the entry, I query `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`; if a match exists, I raise a new `AlreadyInWatchlistError` (mirroring `AlreadyInCollectionError`). I defined that exception class at the top of `watchlist_service.py` and documented it in the function's `Raises:` section.
**How I verified:** I compared line-by-line against `add_to_collection()`: it does the same existence-query-then-raise before insert. The collection also has a DB-level `UniqueConstraint("user_id", "film_id")` on `CollectionEntry`; `WatchlistEntry` has no such constraint, so the service-level check is the sole guard here — the equivalent to what `add_to_collection` relies on at the application layer. The full suite (`pytest tests/ -v`) still passes. The dedup path is exercised by the Comment 3 test work below.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and added `test_add_to_watchlist_nonexistent_film_raises`, modeled on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. I copied the same three fixtures (`app` with an in-memory SQLite DB, `sample_user`, `sample_film`) so the test file stands on its own, then asserted that calling `add_to_watchlist()` with a film_id that isn't in the DB raises `FilmNotFoundError` via `pytest.raises`.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` → 1 passed. Then ran the full suite `pytest tests/ -v` → 5 passed, confirming the rename and dedup changes didn't break anything. The model test was `test_add_to_collection_nonexistent_film_raises`: same fixture setup, same `with app.app_context()` block, same `pytest.raises(FilmNotFoundError)` assertion structure.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the watchlist default — but as a *deliberate, documented* choice rather than an inherited one, paired with clear UX signaling of a list's visibility.

**Reasoning:** CineLog is positioned as a film-tracking *network*, and its differentiating value is social discovery — friends and followers finding films through each other. A watchlist is aspirational, low-sensitivity data ("films I intend to watch"), which is materially less sensitive than viewing history or ratings. Defaulting to public is what makes the discovery/recommendation loop work out of the box: a private-by-default watchlist would require every user to take an explicit action before any social feature produces value, which directly suppresses the network effects the platform is built on. The choice is also cheaply reversible — the existing per-list `public` field lets any user flip an individual list private in one action. The point of this note is that the default is now intentional: I am choosing public, not inheriting it.

**Tradeoff acknowledged:** Public-by-default runs against privacy-by-default / principle-of-least-surprise, which is the safer engineering norm and the expectation under data-protection-by-default regimes (e.g. GDPR). "Low sensitivity" is my assessment, not necessarily every user's — some people treat even a watchlist as private, and for them a public default is an accidental-exposure risk. The private-by-default alternative optimizes for user trust and eliminates that surprise, at the cost of dampening discovery and adding friction for the majority who do want to share. I'm accepting that tradeoff conditionally, with mitigations: (a) show the public/private state prominently at list creation and in the list view, (b) keep the toggle one tap, and (c) revisit if telemetry shows users frequently flipping new lists to private — that would be direct evidence the default is wrong. Any future list types that are inherently sensitive should default private regardless.

## Comment 5 — Sort order
**My position:** Agree with the maintainer — change `get_watchlist()` ordering from alphabetical (`Film.title.asc()`) to date-added, newest first (`WatchlistEntry.date_added.desc()`). Implemented in code.

**Reasoning:** The most common action right after opening a watchlist is "what did I just add / what's new," which recency ordering serves directly. Alphabetical's only real advantage is scanning for one known title in a long list — and that need is better met by an explicit sort/search control later than by making it the default for everyone now.

**Engagement with reviewer's point:** The maintainer's stated reasoning — "most users want to see what they added recently" — is sound and I adopt it. I'll add a codebase-grounded argument they didn't raise: `get_collection()` already sorts by `CollectionEntry.date_added.desc()`. Matching that in `get_watchlist()` gives users one consistent mental model across both sibling features; divergent sort orders between collection and watchlist would be a real usability cost. So this isn't just deferring to the reviewer — recency is independently the better default *and* it makes the two features consistent. If we add sort controls down the line, alphabetical comes back as a user-selectable option rather than the forced default.

## Comment 6 — Rebase
**What conflicted:** I ran `git fetch origin` then `git rebase origin/main`. Two things surfaced:
1. **`.gitignore` (add/add conflict)** — both `main` (`chore: add .gitignore`) and my branch independently added a `.gitignore`. Git flagged an explicit add/add conflict with markers.
2. **`models.py` `WatchlistEntry` silently dropped (the real UUID issue)** — the film-ID refactor on `main` (`refactor: migrate film IDs from integer to UUID`, `07ca580`) changed `Film.id` and `CollectionEntry.film_id` to `String(36)` UUIDs **and removed the `WatchlistEntry` class** (which existed at the common ancestor). Because my feature commits never textually modified that block, the rebase applied `main`'s deletion with *no conflict marker* — `WatchlistEntry` simply vanished from the rebased tree, and `add_to_watchlist()` still referenced integer IDs in its docstring and the route body.

**How I resolved it:**
- `.gitignore`: took the union — kept `main`'s `.pytest_cache/` line alongside my existing entries — then `git add .gitignore` and `git rebase --continue`.
- UUID issue: re-added the `WatchlistEntry` model to `models.py`, this time with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), ...)` to match the UUID `Film.id` (previously `db.Integer`). Updated the now-stale integer references in the watchlist code: `add_to_watchlist()`'s docstring (`film_id (int)` → `film_id (str): UUID of the film`) and the route body comment (`{ "film_id": <int> }` → `<uuid string>`). Committed as `fix: use UUID film_id in watchlist after rebase on main`.

**How I verified no conflict remains:**
- `python -c "from models import WatchlistEntry; print(WatchlistEntry.__table__.c.film_id.type)"` → `VARCHAR(36)`, confirming the FK type now matches `Film.id`.
- Grepped `models.py` / `services/watchlist_service.py` / `routes/watchlist/watchlist.py` for `Integer`/`<int>`/`pre-refactor` — the only remaining `Integer` columns are `Film.year` and `CollectionEntry.rating`, which are genuinely integers, not film IDs.
- `pytest tests/ -v` → 5 passed.
- `git log --merges origin/main..HEAD` → empty (no merge commits); `git log --oneline origin/main..HEAD` shows a clean linear stack of my feature commits on top of `main`.

## Commit History

Rewritten with `git rebase -i` into a clean, conventional, linear stack (no merge commits). `git log --oneline origin/main..HEAD`:

```
56bee42 docs: add pr-response.md with visibility and sort order decisions
01f4d04 test: add test for nonexistent film_id in add_to_watchlist
d952d03 fix: update WatchlistEntry film_id to UUID after main branch refactor
69d8faf fix: sort watchlist by date added, newest first
adc38db fix: add deduplication check to prevent duplicate watchlist entries
fca4f85 fix: rename save_to_watchlist to add_to_watchlist per naming convention
7df7cf2 refactor: use db.session.get for film lookups in services
56893de feat: add watchlist service and add_to_watchlist endpoint
```

> **Screenshot:** paste your `git log --oneline` screenshot here. The block above is the exact output to capture (the top `docs:` commit hash will shift by one short-hash whenever the doc itself is re-amended — the screenshot you take is the source of truth). Eight commits, every message conventional (`feat:`/`refactor:`/`fix:`/`test:`/`docs:`), one logical change each, and no merge commits (`git log --merges origin/main..HEAD` is empty).

---

## PR Description

### What the watchlist feature does
Adds a personal **watchlist** to CineLog — films a user intends to watch, kept separate from their collection (films already watched). It ships a `WatchlistEntry` model plus two endpoints: `POST /watchlist/<user_id>/add` to save a film, and `GET /watchlist/<user_id>` to view the list. Adds are guarded against duplicates and against nonexistent films, mirroring the existing collection service.

### Design decisions
1. **Visibility default — `public=True`.** New watchlists are public by default so social discovery (friends finding films through each other) works out of the box, and the per-list `public` flag lets any user make an individual list private in one action. This is a deliberate, documented choice, not an inherited one — see the Comment 4 entry above for the full tradeoff analysis (privacy-by-default is the safer norm; mitigations and a telemetry-based revisit condition are described there).
2. **Sort order — newest first.** `get_watchlist()` orders by `WatchlistEntry.date_added.desc()` rather than alphabetically, so the most recently added films appear first. This matches `get_collection()`, giving one consistent mental model across both sibling features. Full reasoning is in the Comment 5 entry above.

### How to manually test end to end
1. Install and start the app:
   ```bash
   pip install -r requirements.txt
   python app.py
   ```
   The app runs at `http://localhost:5000` on a local SQLite DB.
2. Create a user and a film to work with (Python shell in the project root):
   ```bash
   python -c "
   from app import create_app, db
   from models import User, Film
   app = create_app()
   with app.app_context():
       u = User(username='alice', email='alice@example.com')
       f = Film(title='Arrival', year=2016, genre='Sci-Fi')
       db.session.add_all([u, f]); db.session.commit()
       print('USER_ID=', u.id); print('FILM_ID=', f.id)
   "
   ```
   Note the printed `USER_ID` and `FILM_ID`.
3. **Add a film to the watchlist** (expect `201` with the new entry, `public: true`):
   ```bash
   curl -X POST http://localhost:5000/watchlist/<USER_ID>/add \
        -H "Content-Type: application/json" \
        -d '{"film_id": "<FILM_ID>"}'
   ```
4. **View the watchlist** (expect `200` with the film, `date_added`, and `public: true`):
   ```bash
   curl http://localhost:5000/watchlist/<USER_ID>
   ```
5. **Verify deduplication** — repeat step 3 with the same `film_id`; the duplicate add is rejected (`AlreadyInWatchlistError`), so the list still contains a single entry.
6. **Verify nonexistent film handling** — repeat step 3 with a random UUID for `film_id`; it returns `404 Film not found` and adds nothing.
7. **Verify sort order** — add a second film, then re-run step 4; the most recently added film appears first.
8. Run the automated suite: `pytest tests/` → 5 passed.
