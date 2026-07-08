# Project 5 — Mixtape Bug Hunt: Submission

**Author:** Mehak Gupta
**Branch:** `bugfix/mixtape`

---

## Milestone 1 — Codebase Map

### Overview

Mixtape is a Flask REST API for a social music app. It follows a strict
**three-layer architecture**:

```
HTTP request → routes/ (parse & format) → services/ (business logic) → models.py (SQLAlchemy ORM) → SQLite
```

The app is assembled by a **factory** (`create_app` in `app.py`), which
registers four blueprints under URL prefixes and calls `db.create_all()`.
There is no authentication layer — the caller passes `user_id` explicitly in
the URL or JSON body.

### Main files and their responsibilities

| File | Role |
|------|------|
| `app.py` | Flask **application factory**. Creates the `db = SQLAlchemy()` singleton, configures the SQLite URI, registers the four blueprints (`/songs`, `/playlists`, `/users`, `/feed`), and creates tables. Must be launched as `FLASK_APP=app:create_app flask run` to avoid a double-import of `db`. |
| `models.py` | All **SQLAlchemy models** and association tables. Defines the data model (below). |
| `seed_data.py` | Drops and recreates the DB, then populates 5 users, 25 songs (with 0 / 1 / 3+ tags), 3 playlists, listening events (recent + old), streaks, and one sample playlist-add notification. Note the comments here: they deliberately seed data that exposes each bug (e.g. multi-tag songs for the search duplicate bug). |
| `routes/songs.py` | Endpoints: search, get song, **rate** a song, **listen** to a song. Delegates to `search_service`, `notification_service.rate_song`, `streak_service.record_listening_event`. |
| `routes/playlists.py` | Endpoints: create playlist, get metadata, **get songs**, **add song**. Delegates to `playlist_service` and `notification_service.add_to_playlist`. |
| `routes/users.py` | Endpoints: get user, **get streak**, **get notifications**, mark notification read. |
| `routes/feed.py` | Endpoints: **friends listening now**, activity feed. Delegates to `feed_service`. |
| `services/streak_service.py` | Records listening events and computes the consecutive-day **listening streak**. |
| `services/feed_service.py` | Builds the "Friends Listening Now" feed (recency-filtered, one row per friend) and the unfiltered activity feed. |
| `services/search_service.py` | Song **search** by title/artist (case-insensitive `ilike`), returns songs + tags. |
| `services/notification_service.py` | Creates & retrieves **notifications**; also owns `add_to_playlist` and `rate_song` (the two friend-interaction actions that should notify a song's sharer). |
| `services/playlist_service.py` | Playlist creation and **ordered song retrieval**. |
| `tests/` | pytest suites for streaks, search, and playlists. |

### Data model (`models.py`)

Seven entities. Three are **association tables** (plain `db.Table`, not model
classes):

- **`User`** — `username`, `email`, `listening_streak` (int), `last_listened_at` (nullable datetime). Has a self-referential many-to-many `friends` relationship via the `friendships` table.
- **`Song`** — `title`, `artist`, `album`, `genre`, `shared_by` (FK → User), `shared_at`. Many-to-many `tags`.
- **`Tag`** — just a `name`.
- **`ListeningEvent`** — one row per play: `user_id`, `song_id`, `listened_at`. This is the source of truth for streaks and the "listening now" feed.
- **`Rating`** — `user_id`, `song_id`, `score` (1–5), with a **unique constraint** on `(user_id, song_id)` so a user has at most one rating per song.
- **`Playlist`** — `name`, `created_by`, `is_collaborative`.
- **Association tables:**
  - `friendships` — symmetric user↔user (seeded in both directions).
  - `song_tags` — song↔tag. **A song with 3 tags produces 3 rows here** — relevant to search.
  - `playlist_entries` — playlist↔song, and crucially carries extra columns: **`position`** (explicit ordering, 1-based), `added_by`, and `added_at`. Songs in a playlist have an explicit position, not just insertion order.

### Data flow trace #1 — Adding a song to a playlist triggers a notification

This is the "working" notification path (Issue #4 references it as the correct pattern to compare against):

1. `POST /playlists/<playlist_id>/songs` with `{song_id, added_by}` hits `add_song()` in `routes/playlists.py`.
2. The route validates the body and calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. `add_to_playlist` loads the `Song`, the adding `User`, and the `Playlist`; appends the song to `playlist.songs` (if not already present) and commits.
4. **If the adder is not the song's original sharer** (`song.shared_by != added_by_user_id`), it calls `create_notification(user_id=song.shared_by, type="song_added_to_playlist", body="…")`.
5. `create_notification` builds a `Notification` row, adds it, and commits.
6. Later, `GET /users/<id>/notifications` → `get_notifications` returns those rows newest-first.

### Data flow trace #2 — Listening to a song updates the streak

1. `POST /songs/<song_id>/listen` with `{user_id}` hits `listen()` in `routes/songs.py`.
2. Route calls `streak_service.record_listening_event(user_id, song_id)`.
3. That function creates a `ListeningEvent` at `now`, then calls `update_listening_streak(user, now)`.
4. `update_listening_streak` compares `now.date()` to `user.last_listened_at.date()`:
   - first ever listen → streak = 1
   - same day → no change
   - exactly 1 day gap → streak += 1
   - larger gap → streak reset to 1
   - then stamps `last_listened_at = now`.
5. `GET /users/<id>/streak` → `get_streak` returns the stored `listening_streak`.

### Patterns I noticed

- **Thin routes, fat services.** Every route does only: parse input → call one service function → `jsonify` the result (or map `ValueError` → 4xx). All business logic lives in `services/`. So every bug traces back to a service file — matching the README.
- **`ValueError` as the not-found / bad-input signal.** Services raise `ValueError`; routes catch it and return 400/404. There are no custom exception types.
- **Datetimes are UTC and sometimes naive.** `datetime.now(timezone.utc)` is used for new rows, but values read back from SQLite come back tz-naive — `streak_service` explicitly re-attaches `timezone.utc` before comparing. Any time/calendar logic has to account for this.
- **Explicit ordering via `position`.** Playlists don't rely on insertion order; they store a `position` column and sort by it.
- **`shared_by` is the notification target.** Both friend-interaction actions (playlist-add, rate) are supposed to notify the *original sharer* of a song, guarded by a "don't notify yourself" check.

*AI disclosure:* I used Claude Code to help read the service files and trace the
two call chains above during orientation; I verified each chain by reading the
route → service → model code directly.

---

## The five issues — reproduction plan

I read all five reports before choosing. My chosen three (each reproducible,
one per service, clear root cause):

- **Issue #1 — Streak resets on Sundays** (`streak_service.py`): `update_listening_streak` skips the increment when `today.weekday() != 6` is false, i.e. on Sundays, so a consecutive-day Sunday listen falls into the reset branch. Reproduce: call `update_listening_streak` with a Saturday→Sunday transition → streak drops to 1.
- **Issue #4 — No rating notification** (`notification_service.py`): `rate_song` saves the rating but never calls `create_notification`, unlike `add_to_playlist`. Reproduce: `POST /songs/<id>/rate`, then check the sharer's notifications — none appear.
- **Issue #5 — Last playlist song missing** (`playlist_service.py`): `get_playlist_songs` returns `songs[:-1]`, dropping the last song. Reproduce: `GET /playlists/<id>/songs` returns 6 for a 7-song playlist.

**Why not Issue #3?** I originally planned to fix Issue #3 (search duplicates),
but it does **not** reproduce in this environment. The `.outerjoin(song_tags)` does
fan out to one raw SQL row per tag (3 rows for a 3-tag song), but `search_service`
uses the legacy `db.session.query(Song).all()` API, which automatically deduplicates
entities by primary key — so the endpoint returns count 1, the correct result. (Only
the newer `session.execute(select(...)).scalars().all()` path returns duplicates.)
Following the milestone's guidance to switch bugs when one can't be reproduced after
a genuine attempt, I swapped Issue #3 for Issue #1.

Stretch (if time): **Issue #2 — feed recency** (`RECENT_THRESHOLD` is a rolling
24 hours instead of "since the start of today", so last night's plays linger).

---

## Milestone 2 — Root Cause Analyses

### Issue #5 — The last song in a playlist never shows up

**How you reproduced it:** GET /playlists/<Friday Energy id>/songs returned count 6
for a playlist seeded with 7 songs. The missing song was always the highest-position
(most recently added) one — "Harlem Renaissance".

**How you found the root cause:** Started at routes/playlists.py get_songs() → it calls
playlist_service.get_playlist_songs(). Read that function top-down: the query correctly
joins playlist_entries, filters by playlist_id, and orders by position ascending, fetching
all 7 rows. The moment I read the return line I saw `songs[:-1]` — that was the exact cause.

**The root cause:** The query returns all songs correctly, but the return statement sliced
the list with `songs[:-1]`, which in Python means "every element except the last." So the
last-positioned song was always discarded before the response was built.

**Your fix and side-effect check:** Changed `songs[:-1]` to `songs`. Verified the endpoint
now returns count 7 with "Harlem Renaissance" present. Checked the empty-playlist case
(returns [], unchanged) and the 1-song case (now returns 1, previously 0), and ran
pytest tests/test_playlists.py.

