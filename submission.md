# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude throughout this project for codebase orientation, tracing call chains between routes and services, and reviewing suspicious code once I'd found it myself. For each bug, I ran the reproduction steps Claude suggested, checked the actual output against my hypothesis, and only looked at the exact line of code after I'd confirmed the bug was real.

A few honest notes on where this worked well and where I had to course-correct:
- I initially ran `curl` inside the `flask shell` Python REPL by mistake — Claude caught it and I switched to a plain terminal.
- When testing the feed fix, my first `curl` still showed the old (buggy) result after editing the file — turned out the Flask dev server doesn't auto-reload with debug mode off, so I had to restart it to pick up the code change.
- For Issue #3 (duplicate search results), I reproduced the raw SQL join and confirmed it does produce 3 duplicate rows for a 3-tag song, but the ORM-level `search_songs()` call and the full test suite both consistently returned only 1 result with no duplicates. Rather than adding a `.distinct()` for a bug I couldn't actually trigger, I left the code as-is and documented this as an investigated-but-not-reproduced finding.
- I made a mistake during an interactive rebase (exited the commit message editor without actually editing the text), which left one commit with its original non-conventional message. I re-ran the rebase to fix it, though a small typo remains in that message.

## Codebase Map

**Main files and their roles:**
- `app.py` — Flask app factory (`create_app`). Initializes the SQLAlchemy `db` instance, registers four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and creates all tables on startup.
- `models.py` — defines the data model: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables (`friendships`, `song_tags`, `playlist_entries`). `playlist_entries` isn't a plain many-to-many join — it carries extra columns (`position`, `added_by`, `added_at`), so a song's place in a playlist is explicit, not just insertion order.
- `routes/` — one blueprint per resource. Each route parses the request (query params or JSON body) and formats the response (`jsonify` + status code), then delegates all actual logic to a function in `services/`.
- `services/` — where the business logic (and all five bugs) live: `streak_service.py`, `feed_service.py`, `search_service.py`, `notification_service.py`, `playlist_service.py`.

**Data flow — a friend rates your song:**
`POST /songs/<song_id>/rate` (`routes/songs.py`) → `notification_service.rate_song(user_id, song_id, score)`. This validates the score is 1–5, looks up the `Song` and `User`, and either updates an existing `Rating` row or creates a new one (there's a unique constraint on `user_id` + `song_id`, so a user can only have one rating per song). Originally, this function committed the rating and stopped — it never notified the song's original sharer, which was Issue #4.

**Pattern noticed:** every route wraps its service call in `try/except ValueError`, converting that into a `400` or `404` response. Services raise `ValueError` for "not found" or "invalid input" conditions instead of returning `None` or an error dict — this convention is consistent across all four route files, which made tracing error paths predictable once I recognized it.

## Root Cause Analysis

### Issue #1 — Listening streak resets on Sunday

**How I reproduced it:** In `flask shell`, created a fresh user with no listening history. Called `update_listening_streak(u, saturday)` with a Saturday timestamp — streak correctly became `1`. Called it again with the very next day (a Sunday) — expected the streak to increment to `2` since the days are consecutive, but it printed `1` instead.

**How I found the root cause:** Opened `services/streak_service.py` and read `update_listening_streak` line by line, following the "one day has passed" branch specifically since that's the consecutive-day case.

**The root cause:** The relevant branch was:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
Python's `datetime.weekday()` returns `6` for Sunday. The extra `today.weekday() != 6` condition meant the streak would only increment on a consecutive day if that day wasn't a Sunday — so any Saturday→Sunday pair fell into the `else` branch and reset the streak to `1`, even though exactly one day had passed and should have counted as consecutive.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause so the branch checks only `days_since_last == 1`. Reran `pytest tests/test_streaks.py` — all 5 tests passed, including same-day, consecutive-day (Sunday included), skipped-day, and new-user cases.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:** Queried a real user's `/feed/<id>/listening-now` endpoint via `curl`. The initial call only showed genuinely recent events (all seeded within the last ~20 minutes), so I manually inserted a `ListeningEvent` for a different user, timestamped ~20 hours before "now" — recent enough to fall inside a 24-hour window, but on the previous calendar day. Querying the feed again showed that user's event, with a `listened_at` timestamp from the day before, appearing alongside a same-day event.

**How I found the root cause:** Opened `services/feed_service.py` and looked at `get_friends_listening_now`.

**The root cause:**
```python
RECENT_THRESHOLD = timedelta(hours=24)
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
```
This is a rolling 24-hour window, not "today." At 2am, the cutoff becomes 2am the previous day — so any event from that point forward counts as "now," even if it happened on a different calendar date.

**My fix and side-effect check:** Changed the cutoff to the start of the current calendar day:
```python
now = datetime.now(timezone.utc)
cutoff = now.replace(hour=0, minute=0, second=0, microsecond=0)
```
Re-ran the same feed query — the previous-day event disappeared, the same-day event remained. Restarted the Flask dev server first, since debug mode is off and code changes don't hot-reload. Also confirmed `get_activity_feed`, which is not filtered by recency, still returns both events — it isn't affected by this change.

---

### Issue #3 — Duplicate search results (investigated, not reproduced)

**How I investigated it:** Called `search_songs("Crown")` against "Crown Heights Anthem," a song seeded with 3 tags. Both the direct call and `pytest tests/test_search.py` consistently returned exactly 1 result, no duplicates.

**How I checked further:** Since the ORM query lacks an explicit `.distinct()`, I expected a 3-row fan-out from the `outerjoin` against `song_tags`. I verified this at the raw SQL level:
```sql
SELECT s.id, s.title, st.tag_id FROM song s
LEFT JOIN song_tags st ON s.id = st.song_id
WHERE s.title LIKE '%Crown%'
```
This returned 3 rows, one per tag — confirming the join itself does fan out as expected.

**Conclusion:** Despite the join producing 3 rows at the SQL level, the ORM query (`db.session.query(Song)...all()`) consistently resolves them into a single `Song` object before conversion to a dict — likely because SQLAlchemy's identity map collapses rows sharing the same primary key. This is on SQLAlchemy 2.0.51. I was not able to reproduce the reported duplicate-results bug in this environment, so I did not make a change to `search_service.py`. Adding a `.distinct()` would be a reasonable defensive change regardless, since the current code relies on ORM behavior rather than being explicit about deduplication.

---

### Issue #4 — No notification when a friend rates your song

**How I reproduced it:** Picked a seeded song and a user who wasn't its sharer. Checked the sharer's notification count (`1`), called `rate_song()` to submit a rating, then checked again — still `1`. No new notification was created.

**How I found the root cause:** Compared `rate_song()` to `add_to_playlist()` in the same file — the latter calls `create_notification(...)` after committing, but `rate_song()` had no equivalent call.

**The root cause:** This wasn't a logic bug but a missing step — `rate_song()` saved the rating and committed, but never notified the song's original sharer. The pattern existed elsewhere in the same file (`add_to_playlist`) but was never applied here.

**My fix and side-effect check:** Added the same notify-the-sharer pattern used in `add_to_playlist`, including the guard against self-notification:
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```
Re-ran the reproduction — notification count went from `1` to `2` after a cross-user rating. Also tested a user rating their own song: notification count stayed the same (`2` → `2`), confirming the self-notification guard works. Also tested rating the same song twice by the same user (an upsert) to make sure nothing crashed — it didn't.

---

### Issue #5 — Last song in a playlist never shows up

**How I reproduced it:** Queried a seeded playlist ("Late Night Vibes") directly against the `playlist_entries` table — confirmed 7 rows. Called `get_playlist_songs()` on the same playlist and got back only 6 songs.

**How I found the root cause:** Opened `services/playlist_service.py` and read the last line of `get_playlist_songs`.

**The root cause:**
```python
return [song.to_dict() for song in songs[:-1]]
```
The query itself correctly returns all songs in position order, but this line slices off the last element before converting to dicts — so the final song in the playlist is silently dropped every time, regardless of playlist size. The function's own docstring claims it "returns all songs in the playlist," directly contradicting this line.

**My fix and side-effect check:** Removed the slice:
```python
return [song.to_dict() for song in songs]
```
Re-ran the reproduction — `get_playlist_songs()` now returns 7, matching the entry count. Ran `pytest tests/test_playlists.py` — all 3 tests passed, including the empty-playlist case, confirming the fix doesn't break behavior when a playlist has zero songs.

## Commits

See `git log --oneline` on `bugfix/mixtape`:
```
d0db1c3 fix: add missing notification when a friend rates your song
b3c884f fix: remove erroneous slice that dropped the last song in a playlist
33708a7 fix: use calendar-day boundary instead of rolling 24h window for listening-now feed
a38787a fix: i removed incorrect sundayexclusion in strak increment logic
```