# Project 5 — Mixtape Bug Hunt: Submission

**Author:** Mehak Gupta
**Branch:** `bugfix/mixtape`


---

## AI Usage

I used Claude Code (Anthropic's CLI) throughout this project, primarily as a
navigation and explanation aid rather than a code generator. Being honest about
where it helped and where it was wrong:

**Codebase orientation.** I asked it to summarize each service file's
responsibility and to trace two call chains (playlist-add → notification, and
listen → streak update). This sped up building my codebase map, but I verified
every claim by reading the route → service → model code myself before writing it
down.

**Where AI pointed me in the WRONG direction (Issue #3).** My original plan was
to fix Issue #3 (search duplicates), and the AI's first prediction was that the
`.outerjoin(song_tags)` would return one duplicate row per tag. When I actually
reproduced it, the endpoint returned `count: 1`, not 3 — the bug did *not* occur.
Digging in, we found the reason: the code uses the legacy `db.session.query(Song)
.all()` API, which auto-deduplicates entities by primary key, so the fan-out is
masked (only the newer `session.execute(select(...))` path would show duplicates).
This is a concrete case where the AI's plausible-sounding diagnosis was wrong
until I verified it by running the code — so I switched Issue #3 for Issue #1.

**Debugging support (Issues #1, #4, #5).** I used AI to confirm the semantics of
`datetime.weekday()` (Sunday = 6) once I had already narrowed Issue #1 to that
comparison, and to compare the `rate_song` vs `add_to_playlist` blocks side by
side for Issue #4. In both cases the AI explained code I had already located; I
confirmed each diagnosis by reading the function and running controlled inputs
(a Flask shell / standalone script) myself before changing anything.

**What I did without AI.** Choosing which three bugs to fix, reproducing each bug,
writing the actual fixes, deciding the boundary/side-effect checks, and running
the test suite were my own work. The AI was a faster way to read and reason about
unfamiliar code, not a substitute for verifying the behavior.


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

I fixed the bugs in the order #5 → #4 → #1 (simplest root cause to most subtle),
and wrote each entry immediately after fixing that bug. Each entry below has all
five required fields.

---

### Issue #5 — The last song in a playlist never shows up

**Issue number and title:** Issue #5 — "The last song in a playlist never shows up"
(reported by darius). Affected service: `playlist_service.py`.

**How I reproduced it:** On a freshly seeded DB, "Friday Energy" is created with 7
songs (`seed_data.py` inserts `all_songs[3:10]` into `playlist_entries` with positions
1–7). I fetched the playlist and counted:
```
GET /playlists/<Friday Energy id>/songs   →   {"count": 6, ...}
```
Six songs came back for a 7-song playlist, and the missing one was always the
highest-`position` entry ("Harlem Renaissance"). This matched darius's report that the
*most recently added* song is the one that disappears, and that adding another song
"frees" the previous one (because the newly added song then takes the last position).

**How I found the root cause:** I followed the call chain top-down rather than jumping to
a guess. Started at the route `get_songs()` in `routes/playlists.py`, which delegates
straight to `playlist_service.get_playlist_songs()`. Reading that function line by line:
the query correctly `.join`s `playlist_entries`, filters by `playlist_id`, and
`.order_by(asc(position))`, fetching all 7 rows. The query was not the problem. The moment
of certainty was the very last line — `return [song.to_dict() for song in songs[:-1]]`.
The `[:-1]` slice is the specific cause, not just a suspicious area.

**The root cause:** The query returns all 7 songs correctly, but the return statement
slices the list with `songs[:-1]`, which in Python means "every element except the last."
Because the list is ordered by `position` ascending, the last element is always the
highest-positioned (most recently added) song, so that one song is silently discarded
before the response is built. The count is therefore always exactly one less than the
real number of songs.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs` so every fetched song is
returned. Verified the same endpoint now returns `"count": 7` with "Harlem Renaissance"
present. Boundary checks on both sides of the slice: an empty playlist still returns `[]`
(previously `[][:-1]` was also `[]`, so no regression), and a 1-song playlist now returns
1 (previously it returned 0 — the same bug at the smallest size). Ran `pytest tests/` —
all 13 tests pass, including `tests/test_playlists.py`.

---

### Issue #4 — Notified on playlist-add but not on rating

**Issue number and title:** Issue #4 — "I got notified when a friend added my song to a
playlist but not when they rated it" (reported by aaliya). Affected service:
`notification_service.py`.

**How I reproduced it:** I compared the two friend-interaction actions on the same shared
song (Crown Heights Anthem, shared by simone). First I read simone's notifications
(`GET /users/<simone>/notifications` → `count: 0`). Then a *different* user (kenji) rated
the song: `POST /songs/<song>/rate` with `{"user_id": <kenji>, "score": 5}` → returned
`201` with the saved rating (score 5). Re-reading simone's notifications → still `count: 0`.
The rating persisted but no notification was ever created — while the playlist-add action
on the same song *does* create one. That contrast confirmed the bug is specific to rating.

**How I found the root cause:** Followed `routes/songs.py rate()` → `notification_service.
rate_song()`. Because both friend-interaction actions live in the same file, I put
`rate_song` and its working sibling `add_to_playlist` side by side. `add_to_playlist` ends
with a guarded `create_notification(...)` call to the song's sharer; `rate_song` validates
the score, upserts the `Rating`, commits, and returns — with no `create_notification` call
anywhere in the function. The confidence came from that line-by-line structural comparison:
the notify step isn't broken, it's absent.

**The root cause:** `rate_song` correctly persists the `Rating`, but it never calls
`create_notification()`. The "notify the original sharer" step that exists in
`add_to_playlist` was simply never written into `rate_song`, so a rating is saved with no
side effect for the sharer. This is a *missing step*, not a faulty comparison or typo —
architectural, exactly as the brief's hint suggested.

**My fix and side-effect check:** Added a `create_notification()` call at the end of
`rate_song` (after the commit), of type `"song_rated"`, guarded by the same
`if song.shared_by != user_id` check `add_to_playlist` uses so a user rating their own song
isn't notified. No new imports were needed — `song`, `rater`, `user_id`, `score`, and
`create_notification` are all already in scope. Verified via a controlled run: a rating by
another user takes the sharer's notifications from 0 → 1 with type `song_rated` and body
`"nova rated your song 'Crown Heights Anthem' 5 stars."`; a user rating their *own* song
adds nothing (guard works). Ran `pytest tests/` — all pass, confirming the existing
playlist-add notification path still works.

---

### Issue #1 — Listening streak resets on Sundays

**Issue number and title:** Issue #1 — "My listening streak keeps resetting" (reported by
kenji). Affected service: `streak_service.py`.

**How I reproduced it:** The buggy function `update_listening_streak(user, now)` takes the
current time as a parameter, so I could reproduce it deterministically without touching the
system clock. I called it with a user whose `last_listened_at` was a Saturday and `now` set
to the following Sunday (consecutive calendar days, starting streak 12): the streak came
back as **1** instead of the expected **13**. A control case with the identical one-day gap
on non-Sunday days (Mon → Tue) correctly returned **13**. Same gap, only the weekday
differed — which isolated "is today Sunday?" as the trigger and matched kenji's report that
"both times it was a Sunday."

**How I found the root cause:** Traced the real flow `routes/songs.py listen()` →
`streak_service.record_listening_event()` → `update_listening_streak()`, then read the
branch that chooses between incrementing and resetting. The `elif` read
`days_since_last == 1 and today.weekday() != 6`. I used AI to confirm the semantics of
`datetime.weekday()` (Monday = 0 … Sunday = 6), then verified by reading the branch myself:
when today is Sunday, `today.weekday() != 6` is False, so the whole `elif` is False and
control falls into the `else`, which resets the streak. The control-vs-Sunday experiment
above proved that was the exact cause, not just a suspicious line.

**The root cause:** Python's `datetime.weekday()` returns 6 for Sunday. The increment
branch required `today.weekday() != 6`, so any consecutive-day listen that landed on a
Sunday failed that condition and fell through to the `else` branch, which sets
`listening_streak = 1`. There is no legitimate reason for consecutive-day streak logic to
care about the day of the week; the `and today.weekday() != 6` clause was spurious and
turned every Sunday into a forced reset.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause so the
branch is simply `elif days_since_last == 1:`. Because this is a boundary condition, I
verified all four cases around the branch: consecutive Sat → Sun now increments (12 → 13),
a same-day repeat listen is unchanged (no double-count), a 2+ day gap still resets to 1,
and a first-ever listen (`last_listened_at is None`) still starts at 1. Ran
`pytest tests/` — all 13 pass, including `tests/test_streaks.py`.
