# Mixtape — Submission

## AI Usage

I used Claude Code (an AI coding agent) throughout this project, mostly for navigation and verification rather than for writing the fixes themselves — the actual fixes were one- or two-line changes once the root cause was located, so the value of AI here was in reading code faster and cross-checking hypotheses, not generating logic.

**Codebase orientation (Milestone 1):** I had Claude read every file in `routes/` and `services/` and summarize what each function is responsible for, then asked it to trace the specific call chain for "a user rates a song" (route → service → model) so I could write the data-flow section of the codebase map without re-deriving it line by line myself. This was accurate and saved real time — the routes are thin enough that a summary is easy to verify against the source in a few seconds.

**Where AI's first read was wrong or incomplete, and I had to verify myself:** For Issue #3 (duplicate search results), reading `search_service.py` alone (with or without AI) suggests a clear bug: `search_songs()` joins `Song` to `song_tags` with no `.distinct()`, so a song with 3 tags should produce 3 duplicate rows. That's the "obviously correct" diagnosis a quick read gives you. I didn't stop there — I actually ran the query (`db.session.query(Song).outerjoin(song_tags, ...).all()`) against real data with a 3-tag song and counted the results. It returned **1** row, not 3, and the project's own `test_search_no_duplicates_multi_tag_song` test passes. It turned out the installed SQLAlchemy version (2.0.51) auto-deduplicates full-entity ORM query results by identity, so the "obvious" bug from static reading doesn't actually reproduce in this environment. If I'd trusted the code-reading diagnosis without running it, I would have "fixed" a bug that wasn't actually observable and possibly missed the real Issue #3 defect. This is the main reason I deprioritized #3 in favor of #1/#4/#5, which I *did* verify reproduce exactly as described, both by direct execution and by the project's existing failing tests.

**Debugging/root-cause work (Milestone 3):** For each bug, I used Claude to do structural comparisons rather than open-ended "find the bug" searches — e.g., for Issue #4, I had it line up `add_to_playlist()` against `rate_song()` (both in `notification_service.py`) side by side and point out the structural difference, which is what confirmed the missing `create_notification()` call was the actual gap and not, say, a routing or filtering issue. For Issue #1, once I'd narrowed the suspicious code to the `elif` branch in `update_listening_streak()`, I asked Claude to explain what `datetime.weekday()` returns for each day of the week, to confirm `6 == Sunday` before concluding the condition was the cause — I verified that against Python's own docs behavior by running it directly rather than taking the explanation at face value.

**What I did NOT rely on AI for:** every root cause conclusion and every fix was verified by actually running code — either the project's pytest suite, a scratch script driving the service function directly with controlled inputs, or live `curl` requests against the running Flask app with real seeded data. AI explanations were a starting point for where to look and how to reason about a function, not the final word on whether a bug existed or was fixed. The Issue #3 case above is the clearest example of why: static/AI-assisted reading pointed at a bug that direct execution showed wasn't actually present given this exact dependency version.

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

---

## Milestone 3: Root Cause Analysis and Fixes

### Issue #1 — Listening streak keeps resetting

**How I reproduced it:** See Milestone 2. Called `update_listening_streak()` directly with a Saturday (`weekday()==5`) then a Sunday (`weekday()==6`) datetime, one calendar day apart — streak stayed at 1 instead of incrementing to 2.

**How I found the root cause:** Started at `services/streak_service.py::update_listening_streak()`, the only place `listening_streak` is mutated (confirmed via the function's own docstring, which is called from `record_listening_event()`, which is called from `POST /songs/<id>/listen` in `routes/songs.py`). The docstring lists exactly three cases: same day → no change, one day later → increment, more than one day → reset. Reading the `if`/`elif`/`else` directly under that docstring, the `elif` branch read `days_since_last == 1 and today.weekday() != 6` — a second condition on `today.weekday()` that isn't mentioned anywhere in the documented rules. That mismatch between the stated contract (three date-gap cases) and the actual code (a fourth, undocumented weekday condition) was the moment I was confident this was the exact cause, not just a suspicious area — the condition has no reason to exist given the function's own spec, and it's an exact match for the reported symptom ("keeps resetting" — specifically on Sundays).

**The root cause:** `days_since_last == 1` correctly detects a consecutive-day listen, but the code additionally required `today.weekday() != 6` (i.e., the *current* day must not be Sunday) before it would increment the streak. `datetime.weekday()` returns `6` for Sunday. So whenever a user's consecutive listen happened to land on a Sunday, the `elif` condition evaluated to `False` even though the day gap was exactly 1, and execution fell through to the `else` branch, which unconditionally resets the streak to 1. The bug wasn't in the date-gap arithmetic (that part was correct) — it was an extra, spec-contradicting condition bolted onto the correct branch.

**Fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` to match the function's own documented contract exactly. Ran `tests/test_streaks.py` — all 5 tests pass, including the previously-failing `test_streak_increments_on_sunday`. Checked both sides of the boundary manually with a scratch script: (a) Saturday → Sunday → Monday, all consecutive, correctly reaches streak 3; (b) Saturday → Monday (skipping Sunday) still correctly resets to 1 — confirming the fix only removed the erroneous weekday check and didn't disturb the still-correct "skip a day → reset" behavior. Ran the full suite (`pytest tests/`) — no other tests affected.

**Commit:** `fix: remove erroneous Sunday check from streak increment logic`

---

### Issue #4 — Rating a song doesn't notify the sharer

**How I reproduced it:** See Milestone 2. Via the live app: `darius` rated nova's seeded song "Midnight Drive" through `POST /songs/<id>/rate` (201, `Rating` created), but `GET /users/<nova_id>/notifications` showed no new notification, staying at the same count as before the rating.

**How I found the root cause:** Started at `routes/songs.py::rate()`, which calls `notification_service.rate_song()` — despite living in `notification_service.py`, the function only validates the score, looks up the song/user, and upserts a `Rating` row; it never calls `create_notification()`. To confirm this was really the gap (and not, say, a missing route or a filter hiding the notification), I compared it directly against `add_to_playlist()` in the same file, which handles the *working* case from the issue description ("notified when a friend added my song to a playlist"). `add_to_playlist()` does the playlist-membership update, then explicitly checks `if song.shared_by != added_by_user_id` and calls `create_notification(...)`. `rate_song()` has the equivalent data available (`song.shared_by`, the rater, the score) but has no analogous block after its `db.session.commit()`. That side-by-side comparison — one function in the "notification service" file that notifies, and a structurally identical one that doesn't — is what confirmed the root cause precisely, rather than just "notifications are broken somewhere."

**The root cause:** `rate_song()` never calls `create_notification()`. The notification-creation step that exists in `add_to_playlist()` (checking `song.shared_by != <actor>` and writing a `Notification` row) was simply never added to the rating code path, even though both actions live in the same file and both are exactly the kind of "friend interacted with your shared song" event the module's own docstring says it handles ("Notifications are generated when friends interact with a user's shared songs").

**Fix and side-effect check:** Added a notification block to `rate_song()`, mirroring the existing pattern in `add_to_playlist()`: after the rating is committed, if `song.shared_by != user_id` (i.e., the rater isn't the song's own sharer), call `create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.")`. Verified via the live app: re-ran the exact reproduction sequence from Milestone 2 — `darius` rates nova's song — and `GET /users/<nova_id>/notifications` now shows a new `song_rated` notification alongside the existing `song_added_to_playlist` one. Checked side effects: re-rated the same song with a different score (upsert path, `existing.score = score`) to confirm the notification still fires correctly on updates, not just first-time ratings, since `rate_song()` doesn't distinguish insert vs. update before this fix either. Also confirmed a user rating their *own* shared song produces no notification (matches the `song.shared_by != user_id` guard, same as `add_to_playlist`'s guard). Ran the full test suite — no existing tests cover this path, so nothing regressed, and no test currently asserts the new behavior (noted as a gap; the three original test files don't cover `notification_service.py`).

**Commit:** `fix: notify song sharer when a friend rates their song`

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** See Milestone 2. DB had 7 rows in `playlist_entries` for the seeded "Late Night Vibes" playlist (positions 1–7), but `GET /playlists/<id>/songs` returned only 6 songs, silently dropping the one at position 7.

**How I found the root cause:** Started at `routes/playlists.py::get_songs()` → `playlist_service.get_playlist_songs()`. Read the query top-down: it joins `Song` to `playlist_entries` on `playlist_id`, filters correctly, and orders by `asc(playlist_entries.c.position)` — the ordering and filtering logic is correct, confirmed by comparing the query against the `playlist_entries` schema in `models.py`. The moment I was confident I'd found the exact cause (not just "somewhere in this function") was the final line: `return [song.to_dict() for song in songs[:-1]]`. The query variable `songs` is the correctly-ordered, correctly-filtered full result set — `songs[:-1]` is a Python slice that unconditionally drops the last element before it's ever serialized. There is no filtering condition, no `LIMIT` in the SQL, and no comment explaining an intentional exclusion — it's a bare off-by-one slice applied after the correct data was already fetched.

**The root cause:** `get_playlist_songs()` builds the fully correct, position-ordered list of songs (`songs`), but then serializes `songs[:-1]` instead of `songs`, discarding the last song in the list every time, regardless of playlist size (a playlist with 1 song returns 0). This directly contradicts the function's own docstring one line above it: "This function returns all songs in the playlist."

**Fix and side-effect check:** Changed `songs[:-1]` to `songs` so the full ordered result set is serialized. Ran `tests/test_playlists.py` — all 3 tests pass, including the two previously-failing ones (`test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`). Checked the boundary explicitly: `test_empty_playlist_returns_empty_list` (0 songs) still passes — confirming the fix doesn't introduce an index error on empty playlists (a plain `songs` with no slicing handles the empty-list case naturally, whereas `songs[:-1]` on an empty list is also safe, but a length-1 playlist under the old code returned `[]` instead of the one song — verified this manually against a fresh 1-song playlist and confirmed it now returns exactly 1 song). Verified live via the app: re-ran the exact reproduction from Milestone 2 against "Late Night Vibes" — `GET /playlists/<id>/songs` now returns `count: 7`, including the previously-missing song at position 7. Checked related functionality: `get_playlist()` (metadata-only) and `get_user_playlists()` don't touch song lists, so they're unaffected. Ran the full test suite — no other tests affected.

**Commit:** `fix: return all songs in playlist instead of dropping the last one`
