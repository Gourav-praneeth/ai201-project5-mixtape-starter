# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude Code (via the IDE) as a pair-programming assistant for essentially the whole project, but in a "drive, don't autopilot" way — I had it do the reading/reproducing/typing, but I made the calls on scope and checked its claims against real output before accepting them.

**Orientation.** Before looking at any issue, I had it read `app.py`, `models.py`, every file in `routes/` and `services/`, and `seed_data.py`, and summarize what each module is responsible for. I also asked it to trace two specific call chains end-to-end — "a user rates a song, from the route down to wherever a notification might get created" and "a user views a playlist's songs, from the route to the DB query" — because I wanted the actual function-by-function path, not just a description of what the feature does. That's what's written up in the Codebase Map section below. This part was straightforward and I didn't have much to double-check — it's just reading code and reporting back accurately, which is a low-risk task for AI to help with.

**Reproducing the bugs.** For each bug, I had it reproduce the reported behavior against the running app and the seeded data *before* touching any code — e.g., hitting `/songs/<id>/rate` and `/users/<id>/notifications` back-to-back to confirm a rating really doesn't produce a notification, or inserting an 8th playlist entry directly into `playlist_entries` to confirm the "always the newest song is missing" behavior darius described. I wanted to see real request/response output, not just "yes this looks like the bug."

**Where the AI's first read was wrong and I had to push back on it.** For Issue #3 (duplicate search results), it initially pointed at the missing `.distinct()` on the `outerjoin` in `search_service.py` as the obvious cause — and it's a real code smell, the raw SQL genuinely returns duplicate rows for a multi-tag song. But when it actually hit the live `/songs/search` endpoint and ran the existing `test_search.py` suite, there were no duplicates at all — every test passed. I made it dig into *why* before accepting either the "it's fixed" or "it's not a bug" conclusion, and it found the real explanation: the installed SQLAlchemy 2.0.51 uses the legacy `Query` API for `db.session.query(...)`, which auto-deduplicates full-entity results by identity even without `.distinct()` — something that isn't obvious from reading the code, only from actually comparing raw SQL row counts against the ORM's `.all()` output. That's a case where I couldn't just take the first explanation at face value; the code-level bug was real but not the live bug in this environment, and I had it swap out Issue #3 for Issue #4 rather than "fixing" something that wasn't actually broken here.

**Root cause write-ups.** After each bug was reproduced and the offending line identified, I had it draft the root-cause-analysis entries below. I checked each one against the actual `pytest` output and curl responses from the reproduction step myself rather than trusting the prose — e.g., confirming `test_streak_increments_on_sunday` really was the one failing test, and that `songs[:-1]` was really the only transformation between the correct query result and the returned list.

**Git cleanup.** After committing, one of my commits ended up bundling three unrelated changes together (the notification fix, the playlist fix, and the submission doc) under a message that only described the notification fix — I hadn't scoped my `git add` carefully. I asked for help splitting it apart without losing anything. It used `git reset --mixed` back to the prior clean commit, re-committed each file separately, and — importantly — showed me `git diff <old-bundled-commit> <new-final-commit>` returning empty before force-pushing, which is how I confirmed no work was actually lost in the split. I made the force-push call myself once I saw that empty diff, since rewriting already-pushed history isn't something I wanted done silently.

Overall the AI was reliable for mechanical work (reading, tracing, running commands, drafting write-ups from confirmed facts) and I had to actually intervene once — Issue #3 — where the first plausible-looking explanation didn't survive contact with the live app.

---

## 1. Codebase Map

### Main files

| File | Responsibility |
|---|---|
| `app.py` | Flask application factory (`create_app`). Configures the DB URI, initializes `flask_sqlalchemy`'s `db`, registers the four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()`. |
| `models.py` | All SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables (`friendships`, `song_tags`, `playlist_entries`). `playlist_entries` is notable — it's a many-to-many table with *extra* columns (`position`, `added_by`, `added_at`), not a plain association table. |
| `routes/songs.py` | `/songs/search`, `/songs/<id>`, `/songs/<id>/rate`, `/songs/<id>/listen`. Thin — parses request data, calls a service, serializes the result. |
| `routes/playlists.py` | `/playlists/` (create), `/playlists/<id>`, `/playlists/<id>/songs` (GET/POST). |
| `routes/users.py` | `/users/<id>`, `/users/<id>/streak`, `/users/<id>/notifications`, `/users/notifications/<id>/read`. |
| `routes/feed.py` | `/feed/<id>/listening-now`, `/feed/<id>/activity`. |
| `services/streak_service.py` | Records listening events and updates a user's daily listening streak. |
| `services/feed_service.py` | Builds the "friends listening now" feed (recency-filtered) and the general activity feed (not recency-filtered, just most-recent-N). |
| `services/search_service.py` | Title/artist substring search over songs, joined against tags. |
| `services/notification_service.py` | Creates notifications and records the two actions that are supposed to trigger them: adding a song to a playlist, and rating a song. Also handles reading/marking-read. |
| `services/playlist_service.py` | Playlist CRUD-ish reads: create, get metadata, get ordered songs, get a user's playlists. |
| `seed_data.py` | Drops and recreates the DB, then seeds 5 users (with friendships), 13 songs (with 0/1/3+ tags to exercise the search bug), 3 playlists, listening events (both "recent" and "older" for the feed bug), and one sample notification. |

**Pattern:** routes are intentionally dumb — they validate the presence of required fields, call exactly one service function, and translate `ValueError` into a 400/404 JSON response. All actual logic (including notification side-effects) lives in `services/`, which is also where all five reported bugs live, matching the README's note.

### Data flow trace: rating a song and triggering a notification

`POST /songs/<song_id>/rate` with `{user_id, score}`

1. `routes/songs.py::rate()` — pulls `user_id`/`score` out of the JSON body, 400s if either is missing, then calls `rate_song(user_id, song_id, int(score))`.
2. `services/notification_service.py::rate_song()`:
   - Validates `1 <= score <= 5`.
   - Looks up the `Song` and the rating `User` (`rater`), 404-equivalent `ValueError` if either is missing.
   - Checks for an existing `Rating` for this `(user_id, song_id)` pair (there's a unique constraint on that pair in `models.py`) — updates it in place if found, otherwise inserts a new `Rating`.
   - Commits.
   - Notably, `rate_song()` does not call `create_notification()` anywhere in its body.
3. Back in the route, the `Rating` is serialized via `.to_dict()` and returned as `201`.

Contrast with `add_to_playlist()` in the same file, which *does* follow through: it appends the song to the playlist, commits, then — if the adder isn't the original sharer — calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)`. `create_notification()` itself is a two-line function: build a `Notification` row, add, commit. The two functions are structurally parallel (validate → mutate → commit) but diverge at this last step.

### Data flow trace: viewing a playlist's songs

`GET /playlists/<playlist_id>/songs`

1. `routes/playlists.py::get_songs()` calls `get_playlist_songs(playlist_id)`.
2. `services/playlist_service.py::get_playlist_songs()` looks up the `Playlist` (404 if missing), then queries `Song` joined to the `playlist_entries` association table, filtered to this playlist, ordered ascending by `position`.
3. The query result is converted to dicts via `[song.to_dict() for song in songs[:-1]]` — note the `[:-1]` slice, which drops the last element of the (correctly ordered) list before returning it.

### Patterns noticed

- **Routes never touch the DB directly except `routes/users.py`'s `get_user()`** (a plain `db.session.get`) — everything else funnels through `services/`.
- **Services communicate errors via `ValueError`**, which every route uniformly converts to a 4xx JSON body. There's no custom exception hierarchy.
- **`to_dict()` on every model** is the sole serialization boundary — services return already-serialized dicts (or full ORM objects for write endpoints), never raw ORM query results, to the routes.
- **Association tables with extra columns (`playlist_entries`) are used both via raw `.insert()` (in `seed_data.py`) and via the ORM `secondary=` relationship (`playlist.songs.append(...)` in `notification_service.add_to_playlist`)** — these two approaches populate the table differently, which matters because `playlist_entries` has non-FK columns (`position`, `added_by`) that only the raw-insert path sets explicitly.

---

## 2. Bug Reproduction (Milestone 2)

### Issues chosen to fix

Originally selected **#1, #3, #5** because those three had existing (informative) test files to anchor against. During reproduction, **#3 turned out not to reproduce** in this environment (see below), so it was swapped for **#4**. Final set: **#1, #4, #5**.

| # | Issue | Reproduced? |
|---|---|---|
| 1 | Streak resets every Sunday | ✅ Yes |
| 2 | Friends Listening Now shows yesterday's listens | Not attempted (deferred) |
| 3 | Duplicate songs in search | ❌ Did not reproduce in this environment — see notes |
| 4 | No notification on rating | ✅ Yes |
| 5 | Last playlist song never shows up | ✅ Yes |

### Issue #1 — streak resets every Sunday

**Root cause:** `services/streak_service.py:73`, inside `update_listening_streak()`:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

The `today.weekday() != 6` clause means: even when the user listened exactly one calendar day after their last listen (the correct condition for incrementing), if *today* happens to be a Sunday the code falls through to the `else` branch and resets the streak to 1 instead of incrementing it.

**How I reproduced it:** Real wall-clock time couldn't be forced to a Sunday, so I called `update_listening_streak()` directly (the same technique the repo's own `tests/test_streaks.py::test_streak_increments_on_sunday` uses) against kenji's actual seeded user row, with hand-built `datetime` objects landing on a real Saturday/Sunday pair:

1. Set kenji's `listening_streak = 11` and `last_listened_at` = a Friday.
2. Called `update_listening_streak(kenji, <Saturday 8pm>)` → streak became **12** (matches kenji's report: "On Saturday night my streak was at 12").
3. Called `update_listening_streak(kenji, <Sunday 9am>)` → streak became **1** instead of the expected **13**.
4. Confirmed via `GET /users/<kenji_id>/streak` → returned `{"streak": 1, ...}`.

Also confirmed by running `pytest tests/test_streaks.py -v`: `test_streak_increments_on_sunday` fails with `assert 1 == 2`; the other four streak tests pass, isolating the bug to the Sunday-specific branch.

### Issue #4 — no notification when a song is rated

**Root cause:** `services/notification_service.py::rate_song()` (lines 73–110) saves/updates the `Rating` row and commits, but never calls `create_notification()`. Its sibling function `add_to_playlist()` does call `create_notification()` after its side effect — `rate_song()` simply never got the equivalent call added.

**How I reproduced it:** Using live seeded data on the running app:

1. `GET /users/<nova_id>/notifications` → `count: 1` (the one seeded "added to playlist" notification).
2. `POST /songs/<midnight_drive_id>/rate` with `{"user_id": "<kenji_id>", "score": 5}` (kenji rating a song nova shared) → `201`, rating saved correctly.
3. `GET /users/<nova_id>/notifications` again → still `count: 1`, identical to before — no rating notification was ever created, even though the rating itself persisted (visible via the song's ratings).

This exactly matches aaliya's report: "rating is saved... but no notification is ever created."

### Issue #5 — last playlist song never shows up

**Root cause:** `services/playlist_service.py::get_playlist_songs()` line 66:

```python
return [song.to_dict() for song in songs[:-1]]
```

`songs` is already correctly ordered by `position`; slicing off the last element unconditionally drops whichever song was added most recently (highest position), no matter how many songs are in the playlist.

**How I reproduced it:** Using the seeded "Friday Energy" playlist (7 songs in the `playlist_entries` table, positions 1–7):

1. `GET /playlists/<friday_energy_id>/songs` → `count: 6`; the 7th song by position ("Harlem Renaissance") is missing.
2. Inserted an 8th entry directly into `playlist_entries` (position 8, a new song, "Midnight Drive") to simulate someone adding a new song.
3. `GET /playlists/<friday_energy_id>/songs` again → `count: 7`; **"Harlem Renaissance" now appears**, but the newly-added "Midnight Drive" (now the highest position) is the one missing.

This exactly matches darius's report: "adding another song 'frees' the previous one and hides the new one instead." (Also matches `tests/test_playlists.py::test_playlist_returns_all_songs`, which fails today asserting `len(songs) == 5` but getting 4.)

### Note: Issue #3 (duplicate search results) did not reproduce

The code has the smell described in the issue — `search_songs()` does an `outerjoin` against `song_tags` and never calls `.distinct()`, so the raw SQL genuinely returns one row per matching tag (verified: a 3-tag song produces 3 raw rows). However, with the installed `SQLAlchemy 2.0.51` + `Flask-SQLAlchemy 3.1.1`, `db.session.query(Song)...all()` is the *legacy* `Query` API, which automatically de-duplicates full-entity results by identity even without an explicit `.distinct()`. I confirmed this two ways:
- Comparing `q.all()` (1 result) against `db.session.execute(q.statement).fetchall()` on the same query object (3 raw rows) for the 3-tag song "Crown Heights Anthem."
- Hitting the live `/songs/search?q=Anthem` and `/songs/search?q=a` (broad match across all 13 seeded songs) endpoints — zero duplicate titles in either case.
- `pytest tests/test_search.py` — all 5 tests pass, including `test_search_no_duplicates_multi_tag_song`.

Per the assignment's guidance, I swapped this issue out for #4 rather than "fixing" code that isn't currently causing the reported symptom in this environment. (The missing `.distinct()` is still worth adding defensively since it's version-dependent behavior, but it isn't one of the three issues I'm treating as reproduced-and-fixed.)

### Additional bug found during reproduction (not one of the 5, not being fixed)

While reproducing Issue #5, `POST /playlists/<id>/songs` returned a `500`. Root cause: `notification_service.add_to_playlist()` does `playlist.songs.append(song)`, relying on the ORM's `secondary=playlist_entries` relationship — but `playlist_entries` has required `position` and `added_by` columns that this relationship doesn't know how to populate, so the resulting `INSERT` violates `NOT NULL constraint failed: playlist_entries.position`. I routed around it (for reproduction purposes only) by inserting into `playlist_entries` directly, the same way `seed_data.py` does. Flagging this here since it currently means **the "add song to playlist" endpoint is completely broken**, independent of Issue #5.

---

## 3. Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How you reproduced it:** Real wall-clock time can't be forced onto a specific weekday, so I called `update_listening_streak()` directly — the same technique the repo's own `tests/test_streaks.py::test_streak_increments_on_sunday` uses — against kenji's real seeded `User` row, with two hand-built, real Saturday/Sunday `datetime` objects. I set `listening_streak = 11` and `last_listened_at` to a Friday, called the function with a Saturday timestamp (streak became 12, matching kenji's "my streak was at 12" on Saturday night), then called it again with a Sunday timestamp the next calendar day. The streak dropped to 1 instead of advancing to 13, which is the exact behavior kenji reported. I also confirmed the result through the real `GET /users/<id>/streak` endpoint and by running `pytest tests/test_streaks.py`, where `test_streak_increments_on_sunday` was the one failing test (`assert 1 == 2`).

**How you found the root cause:** The bug report and the affected-service column in the README both pointed straight at `services/streak_service.py`, and `record_listening_event()` there does nothing but create a `ListeningEvent` and delegate to `update_listening_streak()`, so that's the only function that could own streak math. Reading it top to bottom, every branch looked correct in isolation — new user starts at 1, same-day listens no-op, `days_since_last == 1` increments, anything else resets — until I got to line 73: `elif days_since_last == 1 and today.weekday() != 6:`. The `days_since_last == 1` half of that condition is exactly right for "listened on the very next calendar day," but the `and today.weekday() != 6` clause adds a second, unrelated requirement: it also demands that *today* not be a Sunday. Once I noticed that the whole rest of the function never otherwise refers to a specific day of the week, and that the existing (currently failing) test in `tests/test_streaks.py` is literally named `test_streak_increments_on_sunday`, I was confident this clause — not just "something in the streak logic" — was the entire cause.

**The root cause:** `datetime.weekday()` returns `6` for Sunday. The code at `services/streak_service.py:73` was using `today.weekday() != 6` as an extra guard on the increment branch, meaning: even when `days_since_last == 1` (the sole correct condition for "listened on the next consecutive day"), the streak would only increment if today *isn't* a Sunday. On any Sunday where the user listened exactly one day after their last listen, the `elif` condition evaluated to `False` and execution fell through to the `else: user.listening_streak = 1` branch, discarding the streak. There is no legitimate reason a consecutive-day check should special-case one specific weekday — this clause has no correct purpose and was pure leftover/mistaken logic.

**Your fix and side-effect check:** Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` as the sole condition for incrementing (`services/streak_service.py:73`). This is a one-token-expression deletion — no other branch, function, or caller changes. To check for side effects I ran the full `tests/test_streaks.py` suite (all 5 pass, including the previously-failing Sunday test) and additionally verified both directions of the week boundary by hand: (1) a Sunday→Monday consecutive listen still increments (2 after 1), confirming the fix doesn't accidentally special-case Monday instead; and (2) a multi-day gap that spans a Sunday (Friday → the following Monday, skipping Saturday and Sunday) still correctly resets to 1, confirming the fix didn't loosen the "skipped day" reset path. `record_listening_event()`, `get_streak()`, and the `/users/<id>/streak` route are unchanged and unaffected since they don't touch this conditional.

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How you reproduced it:** On the live app with seeded data: `GET /users/<nova_id>/notifications` returned `count: 1` (the one pre-seeded "added to playlist" notification). I then had kenji rate a song nova shared via `POST /songs/<song_id>/rate` with `{"user_id": "<kenji_id>", "score": 5}`, which returned `201` and a correctly-saved `Rating`. Re-checking `GET /users/<nova_id>/notifications` immediately after still returned `count: 1` — identical to before, with no new entry — even though the rating itself was persisted. This is exactly aaliya's report: "rating is saved... but no notification is ever created."

**How you found the root cause:** The affected-service column pointed at `services/notification_service.py`, which contains both `add_to_playlist()` (the working case aaliya confirmed) and `rate_song()` (the broken case) side by side in the same file. Reading `add_to_playlist()` first established the expected pattern: perform the side effect, commit, then call `create_notification(user_id=song.shared_by, ...)` if the actor isn't the song's own sharer. Reading `rate_song()` immediately below it, the function validates the score, looks up the song and rater, saves or updates the `Rating`, and commits — and then just `return`s. There is no call to `create_notification` anywhere in the function. Confirming `create_notification()` itself works correctly (it's a trivial three-line create/add/commit) and is only ever invoked from `add_to_playlist()` in the whole file made it clear the omission in `rate_song()` was the complete root cause, not a downstream effect of something else.

**The root cause:** `rate_song()` in `services/notification_service.py` (lines 73–110 before the fix) never calls `create_notification()` after saving a rating. Its sibling function, `add_to_playlist()`, performs its side effect and then explicitly notifies `song.shared_by`; `rate_song()` performs its side effect (saving/updating the `Rating` row) and simply returns, with no equivalent notification step ever having been written. The rating data itself is correct and complete — the missing piece is a single function call that was never added.

**Your fix and side-effect check:** Added, immediately after the existing `db.session.commit()` in `rate_song()`, a check mirroring `add_to_playlist()`'s pattern: `if song.shared_by != user_id: create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score} stars.")`. This directly fixes the root cause by giving `rate_song()` the same notify-the-sharer step its sibling already had. To check for side effects I re-ran the live scenario: kenji rating nova's song now produces a `song_rated` notification for nova (notification count went from 1 to 2, with the sibling `song_added_to_playlist` notification still intact and unmodified). I also tested the boundary case of a user rating their *own* shared song — the `!= user_id` guard (matching the existing `!= added_by_user_id` guard in `add_to_playlist`) correctly suppresses a self-notification, and the notification count stayed at 2. Re-rating (updating an existing score) was left firing a fresh notification each time, consistent with `add_to_playlist`'s existing behavior of not deduping notifications either — no new inconsistency introduced.

### Issue #5 — The last song in a playlist never shows up

**How you reproduced it:** Using the seeded "Friday Energy" playlist, which has exactly 7 entries in `playlist_entries` at positions 1–7. `GET /playlists/<friday_energy_id>/songs` returned `count: 6`, with the position-7 song ("Harlem Renaissance") missing from the response despite being present in the table. To confirm the "shifts by one" behavior darius described, I inserted an 8th entry directly into `playlist_entries` (position 8, a new song) — bypassing the separately-broken `POST` endpoint — and re-fetched: the count became 7, "Harlem Renaissance" now appeared, and the newly-added song (now the highest position) became the one missing. That's an exact match for darius's report: "adding another song 'frees' the previous one and hides the new one instead."

**How you found the root cause:** `routes/playlists.py::get_songs()` calls straight into `services/playlist_service.py::get_playlist_songs()`, which is the only function that assembles this response. The SQL query inside it — join to `playlist_entries`, filter by playlist, order ascending by `position` — is correct and returns every row in the right order; I confirmed this by printing the raw `songs` list length before the final line. The very last line of the function, `return [song.to_dict() for song in songs[:-1]]`, was the moment of certainty: `songs` was already complete and correctly ordered, and this line unconditionally throws away its last element via `[:-1]` before serializing. There was no other transformation between the correct query result and the returned list, so this slice was necessarily the entire cause.

**The root cause:** `get_playlist_songs()` in `services/playlist_service.py` (line 66) built the songs list correctly — queried and ordered ascending by `position` — but then returned `songs[:-1]` instead of `songs`, unconditionally dropping the last element of an already-correct, already-ordered list. Because the list is ordered by position ascending, "last element" always means "highest position," i.e. whichever song was most recently added to the playlist. This explains both symptoms in the report: a playlist always appears to be missing exactly one song, and that missing song is always the most recently added one, because the slice re-evaluates against the current (longer) list every time a new song pushes the old "last" song down to second-to-last.

**Your fix and side-effect check:** Changed `songs[:-1]` to `songs` — a one-character-slice removal, no other logic touched. To check for side effects I ran the full `tests/test_playlists.py` suite (all 3 pass, including `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order`, which were previously failing/undertested for this exact case) and additionally checked both boundaries by hand: an empty playlist still correctly returns `[]` (the pre-existing `test_empty_playlist_returns_empty_list` test, unaffected since `[][:-1]` and `[]` are both empty — this boundary was never actually broken), and a playlist with exactly one song — previously the worst case, since `[:-1]` on a single-element list silently returned an empty list — now correctly returns that one song. `create_playlist()`, `get_playlist()`, and `get_user_playlists()` in the same file don't touch `playlist_entries` ordering or slicing and were left unchanged.
