# Mixtape — Submission

## Milestone 1: Codebase Map

### Setup notes

- Created `.venv`, installed `requirements.txt`, ran `python seed_data.py` to populate `mixtape.db`.
- Created branch `bugfix/mixtape`.
- Started the app with `FLASK_APP=app:create_app flask run` and confirmed it responds at `http://127.0.0.1:5000` (verified `GET /songs/search?q=Crown` and `GET /users/<id>` return real seeded data).
- Ran `pytest tests/` before making any changes to establish a baseline: **3 failing, 10 passing**. Failures are `test_streak_increments_on_sunday`, `test_playlist_returns_all_songs`, and `test_playlist_returns_songs_in_order` — these map directly to Issues #1 and #5 below.

### Main files and their responsibilities

- **`app.py`** — Flask application factory (`create_app`). Initializes the shared `db` (Flask-SQLAlchemy) object, sets config (DB URI, secret key), registers the four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()` on startup. This is also why the app must be started with `FLASK_APP=app:create_app flask run` rather than `python app.py` — the `if __name__ == "__main__"` block calls `create_app()` a second time, and combined with how `models.py` imports `db` from `app`, running the module directly double-registers things and trips a SQLAlchemy import error.

- **`models.py`** — All SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables (`friendships`, `song_tags`, `playlist_entries`). Notable design choices:
  - `friendships` is a plain many-to-many table with no ordering; the seed script inserts both directions explicitly to make it symmetric (`add_friendship` inserts `(u1,u2)` and `(u2,u1)`). The `User.friends` relationship is one-directional in its `primaryjoin`/`secondaryjoin` (`user_id == id` / `friend_id == id`), so friendship is only "symmetric" because the seed data makes it so — a friendship added just one way would be visible to only one side.
  - `playlist_entries` is a many-to-many join table with **extra columns** (`position`, `added_by`, `added_at`) beyond the two foreign keys — this is what gives playlists an explicit order instead of relying on insertion order.
  - `song_tags` is a bare many-to-many table (song ↔ tag), no extra columns.
  - Ratings are their own model (`Rating`), not stored on `Song` directly, and are constrained to one rating per `(user_id, song_id)` pair via a unique constraint — so "rating a song twice" is an upsert, not a new row.
  - Every model has a `to_dict()` used directly as the JSON response body — there's no separate serialization layer.

- **`routes/`** — Thin Flask blueprints. Every route: parses the request (query args or JSON body), does minimal presence validation (e.g. "is `user_id` present"), calls exactly one service function, and translates `ValueError` from the service into a `400`/`404` JSON error. No business logic lives here.
  - `songs.py` — search, song detail, rating, and recording a listen.
  - `playlists.py` — create playlist, get playlist metadata, list songs in a playlist, add a song to a playlist.
  - `users.py` — user detail, streak, notifications list, mark-notification-read.
  - `feed.py` — friends-listening-now, activity feed.

- **`services/`** — All business logic. This is where the five tracked bugs live.
  - `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares `now.date()` to `user.last_listened_at.date()`: 0 days → no-op, 1 day → increment, anything else (including "never listened") → reset to 1.
  - `feed_service.py` — `get_friends_listening_now()` pulls each friend's most recent `ListeningEvent` within a 24-hour window (`RECENT_THRESHOLD`), deduplicated to one entry per friend. `get_activity_feed()` is deliberately *not* time-filtered — just the most recent N events from friends.
  - `search_service.py` — `search_songs()` matches title/artist case-insensitively and joins to `song_tags` (to eventually support tag-based search/filtering), then serializes matching songs.
  - `notification_service.py` — `create_notification()` is the generic notification writer. `add_to_playlist()` adds a song to a playlist and notifies the original sharer via `create_notification()`. `rate_song()` upserts a `Rating` row (via the same "check for existing, else insert" pattern as `add_to_playlist`'s song-add) but does not call `create_notification()` anywhere.
  - `playlist_service.py` — `create_playlist()`, `get_playlist()` (metadata only), `get_user_playlists()`, and `get_playlist_songs()` (songs joined through `playlist_entries`, ordered by `position`).

- **`tests/`** — pytest suites for streaks, search, and playlists (three of the five buggy services), each using an in-memory SQLite DB via a `TESTING` app fixture. No test files yet exist for `feed_service.py` or `notification_service.py`'s rating path — those two issues will need new tests written as part of the fix.

- **`seed_data.py`** — Populates the dev DB (`mixtape.db`) with 5 users, symmetric friendships, 13 songs (deliberately split into no-tag / one-tag / three-plus-tag groups to exercise the search dedup bug), 3 playlists, a spread of listening events (some within the last 30 minutes, some 1–14 days old, to exercise the feed's recency filter), and one working "song added to playlist" notification (to show the *correct* pattern side-by-side with the missing rating notification in Issue #4).

### Pattern noticed

Every route is a thin wrapper: parse input → call one service function → serialize output or catch `ValueError` → `4xx`. All business logic, and all five bugs, live one layer down in `services/`. Routes never touch the DB directly except `users.py`'s `GET /users/<id>`, which is the simplest possible read (no service needed since it's just `to_dict()` on the looked-up model). This makes the services the correct (and only) place to look for each bug — consistent with the README's guidance to trace routes back to their service.

### Data flow — a user rates a song

1. Client sends `POST /songs/<song_id>/rate` with JSON body `{"user_id": ..., "score": ...}`.
2. `routes/songs.py::rate()` pulls `user_id` and `score` out of the JSON body, checks both are present, and calls `notification_service.rate_song(user_id, song_id, int(score))`.
3. `rate_song()` (in `notification_service.py`) validates the score is 1–5, confirms the song and user exist, then checks for an existing `Rating` row for `(user_id, song_id)` — if found, updates its `score` in place; otherwise inserts a new `Rating`. Commits and returns the `Rating`.
4. The route serializes the `Rating` via `.to_dict()` and returns `201`.
5. **Contrast with adding a song to a playlist**, which goes `POST /playlists/<id>/songs` → `routes/playlists.py::add_song()` → `notification_service.add_to_playlist()`, which does the playlist-membership update *and* calls `create_notification()` to notify the song's original sharer. `rate_song()` has no equivalent call — this is Issue #4 ("notified when a friend added my song to a playlist but not when they rated it"): the notification-creation step was simply never added to the rating path, even though both live in the same file and both live in `notification_service.py`, so the fix is adding one `create_notification(...)` call, following the exact pattern already used in `add_to_playlist()`.

### The five issues (read in full before starting fixes)

| # | Title | Service | Initial read (to verify during fix milestone) |
|---|-------|---------|-----------------------------------------------|
| 1 | Listening streak keeps resetting | `streak_service.py` | `update_listening_streak()` has a stray `today.weekday() != 6` condition on the "consecutive day" branch — it resets the streak instead of incrementing it whenever the *current* day is a Sunday, even though the user listened on consecutive days. Confirmed failing: `test_streak_increments_on_sunday`. |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | Uses a 24-hour rolling `RECENT_THRESHOLD` from `datetime.now()`, not a calendar-day boundary — needs closer verification of what "yesterday" means to the reporting user (rolling 24h window vs. wall-clock day) before concluding whether this is the actual defect. |
| 3 | Same song shows up twice in search | `search_service.py` | `search_songs()` joins `Song` to `song_tags` without a `.distinct()`/`.group_by()`, so a raw SQL join to a 3-tag song produces 3 rows (confirmed via direct SQL: 3 rows for `Crown Heights Anthem`). However, the installed SQLAlchemy 2.0.51 ORM `Query.all()` auto-deduplicates full-entity results by identity, so this **does not currently reproduce** as a duplicate through `search_songs()` in this environment (verified empirically: `search_songs` returns 1 result, and the existing `test_search_no_duplicates_multi_tag_song` test passes). Needs more investigation — either the bug manifests in a different code path/version than tested here, or the real defect is elsewhere (e.g. filtering/sorting by tag) and needs re-examination against the full issue description. |
| 4 | Rating a song doesn't notify the sharer, but adding to a playlist does | `notification_service.py` | Confirmed by reading the code: `add_to_playlist()` calls `create_notification()`; `rate_song()` never does. Straightforward fix — mirror the existing pattern. |
| 5 | Last song in a playlist never shows up | `playlist_service.py` | `get_playlist_songs()` returns `songs[:-1]` instead of `songs` — an off-by-one slice that drops the last song after ordering by position. Confirmed failing: `test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`. |

### Rough plan for which three to fix first

Issues **#1**, **#4**, and **#5** already have clear, single-root-cause fixes confirmed by reading the code (and, for #1 and #5, by failing tests). Issue **#3** needs more investigation before I can be confident about the actual fix in this SQLAlchemy version, and issue **#2** needs a clearer read of what "yesterday" means for the reporting user before changing the recency window. I'll start with #1, #4, #5, then revisit #2 and #3 if time allows.

---

## Milestone 2: Bug Reproduction

Reproduced each of the three chosen bugs before writing any fix code. No code was changed in this milestone — only reproduction scripts/requests run against the seeded dev DB and a scratch in-memory DB.

### Issue #1 — Listening streak resets on Sunday

**How I reproduced it:** Today's real date (2026-07-03) is a Friday, so the buggy `today.weekday() != 6` branch can't be hit through the live app's wall-clock `datetime.now()`. Instead I called the actual service function, `update_listening_streak()` (the same function `record_listening_event` calls from the live `/songs/<id>/listen` route), directly against a fresh in-memory DB, supplying two controlled UTC datetimes: Saturday 2026-07-04 12:00 (`weekday() == 5`) and Sunday 2026-07-05 12:00 (`weekday() == 6`), one day apart.

- Listen on Saturday → `listening_streak == 1` (correct, first listen).
- Listen on Sunday (the very next day) → `listening_streak` stayed at **1** instead of incrementing to **2**.

This confirms the condition `elif days_since_last == 1 and today.weekday() != 6` in `services/streak_service.py` falls through to the `else` reset branch specifically when the *current* day is a Sunday, regardless of the day actually being consecutive. **Data condition needed:** `last_listened_at` one calendar day before `now`, and `now`'s weekday is Sunday. This matches the existing (already-failing) test `test_streak_increments_on_sunday`.

### Issue #4 — Rating a song doesn't notify the sharer

**How I reproduced it:** Started the app (`FLASK_APP=app:create_app flask run`) against freshly seeded data, and used two seeded users with a real sharer relationship: `nova` (id `489847de-...`) shared the song "Midnight Drive" (id `78d9724e-...`), and `darius` (id `ee08607c-...`) is her friend.

1. `GET /users/<nova_id>/notifications` → baseline: 1 notification (the seeded "song added to playlist" one), count = 1.
2. `POST /songs/<song_id>/rate` with `{"user_id": "<darius_id>", "score": 5}` → `201`, a `Rating` row is created successfully.
3. `GET /users/<nova_id>/notifications` again → still count = 1, **no new notification appeared**.

**Contrast:** the seeded notification with `notification_type: "song_added_to_playlist"` proves the notification pipeline itself works — `add_to_playlist()` calls `create_notification()`. The rating action goes through `rate_song()` in the same file, which never calls `create_notification()`. **Sequence needed:** any user other than the song's sharer submits a rating for that song — the sharer should get notified but doesn't.

### Issue #5 — Last song in a playlist never shows up

**How I reproduced it:** Using the seeded playlist "Late Night Vibes" (id `a8163c90-...`), queried the `playlist_entries` table directly to establish ground truth: 7 rows exist for this playlist, at positions 1 through 7.

- `GET /playlists/<playlist_id>/songs` → returned `"count": 6`, containing songs at positions 1–6 only. The song at **position 7** ("Frequencies" by Static Era) was silently missing from the response, with no error.

**Data condition needed:** any playlist with at least one song — `get_playlist_songs()` in `services/playlist_service.py` orders songs by position correctly but then returns `songs[:-1]`, unconditionally dropping the last element of the ordered list before serializing. This matches the existing (already-failing) tests `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order`.

### Notes

- After reproduction, re-ran `python seed_data.py` to reset the dev DB to a clean seeded state before starting fix work, since the Issue #4 repro step wrote a real `Rating` row to `mixtape.db`.
- No code changes were made in this milestone.
