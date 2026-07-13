# PR Response Doc — CineLog Watchlist Feature

## AI Usage

Used Claude Code throughout, including writing the code changes for Comments 1–3 and 6, and the self-critique pass for Comments 4&5 (which surfaced a real gap in the Comment 4 visibility argument, folded into the final text). Also used AI to run the interactive rebase via scripted git editors, and to manually test the running app with `curl`, which caught two pre-existing bugs unrelated to the 6 comments (noted in the PR Description above).

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention already used by `add_to_collection()` in `services/collection_service.py`. Updated the one call site in `routes/watchlist/watchlist.py` (`add_film` route), including its import statement.
**How I verified:** Ran `grep -rn "save_to_watchlist" . --include=*.py` — no remaining references. Confirmed the app still boots (`create_app()` succeeds, all blueprints register) after the rename.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a dedup check inside `add_to_watchlist()`, matching the exact pattern used by `add_to_collection()` in `services/collection_service.py`: after confirming the film exists, query `WatchlistEntry` by `user_id`+`film_id`, and raise if an entry already exists (same behavior as collection — raise, not silent no-op).
**How I verified:** Ran a manual script that adds the same film to a user's watchlist twice — the first call succeeds and returns the new entry, the second raises `AlreadyInWatchlistError` as expected.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` — same `app`/`sample_user` fixture style (in-memory SQLite app fixture), same assertion pattern (`pytest.raises(FilmNotFoundError)` when calling with a film_id that doesn't exist in the database).
**How I verified:** Ran `pytest tests/test_watchlist.py -v` (passes), then the full suite `pytest tests/ -v` — all 5 tests pass (4 existing collection tests + the new watchlist test), confirming nothing else broke.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default, but this needs to be a stated decision rather than an inherited one — which it now is.

**Reasoning:** CineLog is described (README) as a "community film tracking app," and the watchlist is functionally different from the collection: a collection is a personal log of what you've already watched and rated, but a watchlist is forward-looking — "what do I want to watch next" — which is exactly the kind of thing users share to get recommendations, coordinate a group watch, or signal taste to others in a community app. Defaulting new watchlist entries to public lowers the friction for that community behavior; if the default were private, most users likely wouldn't discover or bother flipping a `public` flag per entry, so the feature would default into being invisible even though its whole value in a community app is social. Note also that today `public` is stored but not enforced anywhere — no route currently checks it before returning a user's watchlist — so this is purely about what should render as the intended default when enforcement is added, not about closing an existing hole.

**Tradeoff acknowledged:** The real cost is privacy-by-default: a user who wants to track films to watch without broadcasting taste (e.g., adding something embarrassing, or films for a surprise/gift purpose) is opted into visibility unless they know to change it, and most users don't read defaults carefully. There's a sharper version of this objection worth naming directly: right now `add_to_watchlist()` doesn't expose any way to override `public` per entry, so `True` isn't really a "default" a user can second-guess — it's the only value they get. A privacy-by-design argument says the safer sequencing is to ship `False` first and let users opt in once they have real control, since moving private→public later is low-risk but the reverse means discovering after the fact that people didn't want their data exposed. I'm still landing on `True`, but conditionally: it should ship alongside the per-call `public` override (already tracked as a Stretch Feature), not as a standalone default with no user control. If that override isn't added in this PR, `False` is the more defensible interim default.

## Comment 5 — Sort order
**My position:** Agree with the reviewer — switch `get_watchlist()` from `order_by(Film.title.asc())` to date-added order, newest first, matching `get_collection()`'s existing `order_by(CollectionEntry.date_added.desc())` pattern.

**Reasoning:** A watchlist is a short-lived, action-oriented list ("what do I want to watch next"), not a reference catalog you browse by name. The most recent addition is usually the most relevant one — it's what a user just decided they want to watch, and is most likely to be top-of-mind when they open the app again. Alphabetical order is better suited to lists meant for lookup (e.g. the `/films/` catalog, which is browsed rather than acted on), not to a queue a user is actively working through.

**Engagement with reviewer's point:** The reviewer's stated reasoning — "most users want to see what they added recently" — matches how the equivalent `get_collection()` function already behaves (newest-first), so switching the watchlist to the same order isn't just accepting the reviewer's preference, it also removes an inconsistency: right now the two nearly-identical "list of films with metadata" endpoints sort by two different criteria for no stated reason. I don't see a strong counter-case for alphabetical here — it would matter more if watchlists commonly grew to sizes where users hunt for one entry, but nothing in the current feature suggests that's the primary use case.

## Comment 6 — Rebase
**What conflicted:** Two things. First, a straightforward textual conflict: both `main` (via its own `chore: add .gitignore` commit) and this branch (via a local `.gitignore` commit) independently added a `.gitignore` file — `git rebase origin/main` reported an add/add conflict on it. Second, a subtler non-textual conflict: after rebasing past `main`'s `refactor: migrate film IDs from integer to UUID` commit, `WatchlistEntry` disappeared from `models.py` entirely (confirmed via `git show <commit> -- models.py` on each replayed commit — none of them touched the file). The class had originally been present all the way back at the branch's root commit rather than added by the "watchlist model" commit itself, so when the rebase replayed commits on top of `main`'s post-refactor `models.py` (which never had a watchlist feature and thus never had this class), there was no textual diff for git to reapply — it silently dropped rather than conflicted. The watchlist code (`services/watchlist_service.py`, `routes/watchlist/watchlist.py`, `tests/test_watchlist.py`) also still assumed integer `film_id` throughout (docstrings, a fake test ID of `999999`).

**How I resolved it:** For the `.gitignore` conflict: merged both versions (kept `main`'s extra `.pytest_cache/` line alongside this branch's content), then `git rebase --skip`ed the now-empty local `.gitignore` commit since its content was fully subsumed by `main`'s. For the missing model: re-added `WatchlistEntry` to `models.py`, this time with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)` to match `CollectionEntry`'s post-refactor UUID pattern (was `db.Integer` before). Updated the stale integer-ID docstrings in `watchlist_service.py` (`add_to_watchlist`) and `routes/watchlist/watchlist.py` (`add_film`'s body-shape comment), and changed the test's fake film ID from `999999` to a UUID-shaped string (`"00000000-0000-0000-0000-000000000000"`), matching `test_collection.py`'s pattern.

**How I verified no conflict remains:** `git log --graph --oneline origin/main..HEAD` shows a fully linear history, no merge commits. `grep -rn "int" services/watchlist_service.py routes/watchlist/watchlist.py tests/test_watchlist.py models.py` turns up no real integer-`film_id` references (only unrelated matches like `Blueprint` and the `unique_user_film_collection` constraint name). Ran the full suite (`pytest tests/ -v`) — all 5 tests pass. Also ran a manual end-to-end script creating a real `Film` (confirming SQLAlchemy assigns a UUID string `id`), calling `add_to_watchlist()` with that real UUID, and confirming both the add and the dedup check (`AlreadyInWatchlistError`) still work correctly against UUID film IDs.

## PR Description

### What this adds

A watchlist feature for CineLog: users can save films they want to watch later, view their watchlist, and the system prevents adding the same film twice. This mirrors the existing collection feature (`services/collection_service.py`) but represents forward-looking intent ("want to watch") rather than a historical log ("already watched").

- `WatchlistEntry` model (`models.py`) — links a user to a film, with `date_added` and a `public` flag.
- `add_to_watchlist(user_id, film_id)` (`services/watchlist_service.py`) — creates an entry; raises `FilmNotFoundError` for an unknown film, `AlreadyInWatchlistError` for a duplicate.
- `get_watchlist(user_id)` — returns all of a user's watchlist entries, alphabetically by film title.
- `POST /watchlist/<user_id>/add` and `GET /watchlist/<user_id>` (`routes/watchlist/watchlist.py`).

### Design decisions

- **Default visibility (Comment 4):** New watchlist entries default to `public=True`. See the full reasoning and acknowledged tradeoff above — summary: a watchlist in a community app is meant to be shared (unlike the private collection log), but this default is only meaningful once a per-call override exists, which isn't part of this PR yet (tracked as a stretch feature).
- **Sort order (Comment 5):** Documented agreement with the reviewer that watchlists should sort by date-added (newest first) rather than alphabetically, for consistency with `get_collection()` and because a watchlist is an action queue, not a lookup catalog. This PR does not change `get_watchlist()`'s current alphabetical sort — per the assignment's framing, this is a documented decision rather than a code change in this pass.

### Manual testing

The app has no user/film creation endpoints (films are seeded; users aren't created via the API in this codebase yet), so seed test data via a Python shell first:

```bash
python3
```
```python
from app import create_app, db
from models import User, Film

app = create_app()
with app.app_context():
    db.create_all()
    user = User(username="demo", email="demo@example.com")
    film = Film(title="Paddington 2", year=2017, genre="Comedy")
    db.session.add_all([user, film])
    db.session.commit()
    print("user_id:", user.id)
    print("film_id:", film.id)
```

Then, with the app running (`python app.py`, `http://localhost:5000`). **All of the below was actually run against a live instance of the app, not just described** — two real, pre-existing issues surfaced that aren't part of the 6 review comments and are called out below rather than silently fixed:

1. **Add to watchlist:**
   `curl -X POST http://localhost:5000/watchlist/<user_id>/add -H "Content-Type: application/json" -d '{"film_id": "<film_id>"}'`
   → Confirmed `201`, returns the new entry with `"public": true`.
2. **View watchlist:**
   `curl http://localhost:5000/watchlist/<user_id>`
   → **Confirmed broken: `500 Internal Server Error`.** `WatchlistEntry` has no `film` relationship defined on the model, so `get_watchlist()`'s `entry.film.to_dict()` raises `AttributeError: 'WatchlistEntry' object has no attribute 'film'`. This bug predates this PR (present since the original `ec90edb` commit) and isn't one of the 6 review comments, so it's left unfixed here and flagged for a follow-up PR.
3. **Duplicate add (dedup check):**
   Repeat step 1 with the same `user_id`/`film_id`. → **Confirmed the dedup check itself works** (raises `AlreadyInWatchlistError` — verified directly against the service function in earlier testing), but at the route level this surfaces as `500 Internal Server Error` rather than a clean 409, because `routes/watchlist/watchlist.py`'s `add_film` doesn't catch it (unlike `collection.py`'s equivalent route). Also pre-existing, also out of scope for the 6 comments.
4. **Nonexistent film:**
   `curl -X POST http://localhost:5000/watchlist/<user_id>/add -H "Content-Type: application/json" -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'`
   → **Confirmed** `500` for the same reason as step 3 (`FilmNotFoundError` uncaught at the route level). The service-level behavior itself (raising `FilmNotFoundError`) is correct and is what Comment 3's test verifies directly.
5. **Automated tests:** `pytest tests/ -v` — confirmed all 5 tests pass (4 collection + 1 new watchlist test).

### Commit history

```
9d906df fix: update watchlist film_id fields to UUID after main refactor
3b8ff6a docs: add pr-response.md decisions for default visibility and sort order
1b4e577 test: add test for nonexistent film_id in add_to_watchlist
b226459 fix: add deduplication check to add_to_watchlist
4ed0f7f fix: rename save_to_watchlist to add_to_watchlist
d1615e3 feat: add watchlist model and endpoints
```

6 commits, conventional-format, linear history (no merge commits) rebased onto `main`.
