#  submission.md — Mixtape starter

---

# AI Usage Section

I used AI tools as a debugging assistant to help trace execution flows, understand SQLAlchemy behavior, and identify possible root causes of bugs.

## How I used AI:
- Traced request flow across routes and services (feed, playlist, songs)
- Understood SQLAlchemy join behavior in search and playlist queries
- Identified likely causes of duplication, missing data, and logic errors
- Used AI to summarize service responsibilities and system structure

## What I verified manually:
- Confirmed bugs by reproducing API behavior step-by-step
- Traced function calls directly in the codebase
- Validated fixes by checking API responses after changes

## Where AI was incomplete:
- AI initially suggested rolling 24-hour logic for feed, but I verified calendar-day logic was required
- Some explanations required manual confirmation of DB join behavior and notification flow

---

# Codebase Map 

##  Project Overview

Mixtape is a Flask + SQLAlchemy backend for a social music app where users can:
- listen to songs
- create playlists
- rate songs
- view activity feeds
- receive notifications

Architecture follows a service-layer design:
- Routes handle HTTP requests
- Services contain business logic
- Models define database schema

---

## Main Files and Responsibilities

### app.py
- Initializes Flask app
- Configures SQLAlchemy database
- Registers all blueprints

---

### models.py
Defines database models:
- User → profile, friendships, streak tracking
- Song → metadata and ownership
- Playlist → collections of songs
- playlist_entries → join table with ordering
- ListeningEvent → activity log for feeds
- Notification → user notifications
- Rating → song ratings
- Tag → song tags

---

## Routes Layer

### songs.py
- search songs
- get song details
- rate songs
- record listening events

### playlists.py
- create playlist
- get playlist details
- add songs to playlist

### feed.py
- friends listening now feed
- activity feed

### users.py
- user profile
- streak tracking
- notifications

---

## Services Layer

- streak_service → streak logic
- feed_service → feed generation
- search_service → song search
- playlist_service → playlist operations
- notification_service → notifications + ratings

---

## Data Flow Example — Listening → Feed

POST /songs/<song_id>/listen

- routes/songs.py → record_listening_event()
- Creates ListeningEvent
- Updates streak

GET /feed/<user_id>/listening-now

- routes/feed → get_friends_listening_now()
- Above function in feed_service.py queries events
- Returns friend + song data

---

## Data Flow Example — Playlist → Notification

POST /playlists/<playlist_id>/songs
- add_to_playlist() called
- Song added to playlist
- If not original sharer → notification created
- Notification visible via /users/<user_id>/notifications

---

# Bug Fix Reports

---
## Issue #1 — Listening streak resets incorrectly (`streak_service.py`)

### How I reproduced it

I reproduced the bug by testing the listening streak feature through the API.

**Request:**

```http
POST /songs/<song_id>/listen
```

**Request Body:**

```json
{
  "user_id": "841520f8-8825-4c7e-9c1d-88a48417e98c"
}
```

### Steps:

1. Sent a listen request for a user and verified that the streak started at `1`.
2. Simulated a consecutive-day listen by updating the listening timestamp.
3. Sent another listen request for the same user on the next calendar day.
4. Checked the user's streak value.

### Actual behavior:

The streak remained:

```
Day 1 → streak = 1
Day 2 → streak = 1
```

The streak did not increment even though the user listened on consecutive days.

### Expected behavior:

The streak should:

```
First listen → streak = 1
Consecutive day listen → streak increases by 1
Skipped day → streak resets to 1
```

---

### How I found the root cause

I traced the request flow through the application:

```
routes/songs.py
        |
        | POST /songs/<song_id>/listen
        ↓
services/streak_service.py
        |
        ↓
record_listening_event()
        |
        ↓
update_listening_streak()
```

I inspected the `update_listening_streak()` function because it contains the logic that decides whether the streak should increment or reset.

The moment I identified the issue was when I found this condition:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

The weekday check was unrelated to whether two listening events happened on consecutive calendar days.

I verified this by testing a consecutive-day listen that crossed a Sunday boundary, where the streak failed to increment.

---

### Root cause

The streak calculation incorrectly depended on the day of the week:

```python
today.weekday() != 6
```

This condition prevented streak increments on Sundays.

The feature requirement is based on **calendar-day difference**, not weekdays. A user listening on consecutive days should increase the streak regardless of whether the day is Monday, Sunday, or any other day.

The incorrect weekday condition caused valid consecutive listens to be treated as a streak reset.

---

### Fix and side-effect check

### Fix:

Removed the weekday dependency and used only the number of days between listening events.

The updated logic checks:

```python```
days_since_last == 1
```

to determine whether the user listened on the previous calendar day.

### Why this fixes the issue:

- Consecutive-day listens correctly increment the streak.
- Skipped days still reset the streak to `1`.
- Multiple listens on the same day do not increase the streak.

---

### Side-effect checks performed:

I verified related streak behaviors after the fix:

✅ Tested weekday-to-weekday listening:

```
Monday → Tuesday
```

Result:
```
streak increased correctly
```

✅ Tested weekend boundary:

```
Saturday → Sunday
```

Result:
```
streak increased correctly
```

✅ Tested skipped day scenario:

```
Monday → Wednesday
```

Result:
```
streak reset to 1
```

✅ Tested multiple listens on the same day:

Result:
```
streak remained unchanged
```

The fix corrected the streak calculation without affecting other listening-event behavior.

---

## Issue #2 — Friends Listening Now shows people from yesterday (`feed_service.py`)

### How I reproduced it

1. Started the application and created test users with friendship relationships.

2. Created listening events for a friend using:

```http
POST /songs/<song_id>/listen
```

Request body:

```json
{
  "user_id": "841520f8-8825-4c7e-9c1d-88a48417e98c"
}
```

3. Added test data where a friend listened to a song yesterday but the event was still within the last 24 hours.

4. Requested the feed using:

```http
GET /feed/<user_id>/listening-now
```

5. Observed that the response included the friend's listening activity from yesterday.

### Actual behavior

The "Friends Listening Now" feed displayed listening events that happened yesterday if they were within the previous 24 hours.

Example:

```
Current time: July 7, 10:00 AM

Listening event:
July 6, 11:00 AM

Difference:
23 hours ago
```

The event appeared in the feed even though it was from the previous calendar day.

### Expected behavior

The "Friends Listening Now" feed should only show activity from the current calendar day.

Expected:

- Include events from today
- Exclude events from yesterday, even if they are within 24 hours

---

### How I found the root cause

I traced the request flow starting from the API endpoint:

```
routes/feed.py
        |  GET /feed/<user_id>/listening-now
        v
services/feed_service.py
        |
        v
get_friends_listening_now(user_id)
```

I inspected the filtering logic inside `get_friends_listening_now()`.

The function used:

```python
RECENT_THRESHOLD = timedelta(hours=24)
```

and calculated the cutoff:

```python
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
```

The database query filtered events using:

```python
ListeningEvent.listened_at >= cutoff
```

The moment I became confident this was the cause was when I tested an event from yesterday that was only 23 hours old. The query correctly included it because it satisfied the 24-hour condition, but it violated the feature requirement of showing only today's activity.

---

### Root cause

The root cause was the use of a rolling 24-hour time window instead of a calendar-day comparison.

The code:

```python
datetime.now(timezone.utc) - timedelta(hours=24)
```

calculates a timestamp exactly 24 hours before the current moment.

This means activity from yesterday can still appear if it happened less than 24 hours ago.

The feature requirement was based on the calendar date, not elapsed hours.

---

### Fix and side-effect check

### Fix

Changed the filtering logic to compare calendar dates instead of using a rolling 24-hour timestamp.

The updated logic checks whether:

```python
event.listened_at.date() == today.date()
```

This ensures only events from the current day are returned.

### Why this fixes the issue

The feed now evaluates:

```
Same calendar day?
        |
        +-- Yes → include event
        |
        +-- No → exclude event
```

instead of:

```
Within last 24 hours?
        |
        +-- Yes → include event
```

which incorrectly included previous-day activity.

---

### Side-effect checks performed

After applying the fix, I verified:

1. Today's listening activity still appears:

```
Today 9:00 AM listening event
        ↓
Appears in Friends Listening Now feed
```

2. Yesterday's activity is removed:

```
Yesterday 11:00 PM listening event
        ↓
Does not appear in Friends Listening Now feed
```

3. Feed ordering was unchanged:
- Most recent listening events still appear first.

4. The activity feed endpoint was checked separately:

```http
GET /feed/<user_id>/activity
```

and confirmed it still returns recent friend activity without being affected by the "listening now" filtering change.

---

## Issue #3 — The same song keeps showing up twice in search (`search_service.py`)

### How I reproduced it

1. Started the Flask application and loaded the provided seed data using:

```bash
python seed_data.py
```

2. Reviewed the seeded songs and identified songs with multiple tags.

Example seed data:

```python
("Crown Heights Anthem", "Borough Kings", "rap",
 ["rap", "hip-hop", "boom bap"])
```

This song has three tag relationships:

```
Crown Heights Anthem
    |
    ├── rap
    ├── hip-hop
    └── boom bap
```

3. Called the search endpoint:

```http
GET /songs/search?q=rap
```

4. Observed that songs with multiple tag matches could appear more than once in the response.

Example incorrect response:

```json
{
    "results": [
        {
            "title": "Crown Heights Anthem",
            "artist": "Borough Kings"
        },
        {
            "title": "Crown Heights Anthem",
            "artist": "Borough Kings"
        }
    ],
    "count": 2
}
```

The database contains only one `Song` record for "Crown Heights Anthem", so duplicate results indicate a query issue.

Expected behavior:

```json
{
    "results": [
        {
            "title": "Crown Heights Anthem",
            "artist": "Borough Kings"
        }
    ],
    "count": 1
}
```

---

### How I found the root cause

I traced the request flow from the API endpoint:

```
routes/songs.py
        |
        | GET /songs/search?q=rap
        |
        v
services/search_service.py
        |
        | search_songs()
        |
        v
models.py
        |
        | Song + song_tags relationship
```

The search route calls:

```python
results = search_songs(query)
```

in:

```
services/search_service.py
```

I then inspected the SQLAlchemy query:

```python
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(
        db.or_(
            Song.title.ilike(f"%{query}%"),
            Song.artist.ilike(f"%{query}%"),
        )
    )
    .all()
)
```

The moment I became confident this was the cause was when I connected the query with the seed data:

- `Crown Heights Anthem` has three tags.
- The query joins through `song_tags`.
- Each tag relationship creates another joined row for the same song.

The SQL join result can look like:

```
Song                  Tag
--------------------------------
Crown Heights Anthem  rap
Crown Heights Anthem  hip-hop
Crown Heights Anthem  boom bap
```

The same song is returned multiple times because the join creates multiple matching rows.

---

### The root cause

The search query joins the many-to-many `song_tags` table:

```python
.outerjoin(song_tags, Song.id == song_tags.c.song_id)
```

Songs with multiple tags create multiple database rows during the join.

Because the query does not remove duplicate song records, the same song can appear multiple times in the API response.

The problem is caused by the SQLAlchemy query behavior in `search_service.py`, not by the route or the seed data.

---

### Fix and side-effect check

### Fix:

Updated the query to return unique songs by adding:

```python
.distinct()
```

Updated query:

```python
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(
        db.or_(
            Song.title.ilike(f"%{query}%"),
            Song.artist.ilike(f"%{query}%"),
        )
    )
    .distinct()
    .all()
)
```

---

### Why this fixes the root cause

`DISTINCT` removes duplicate rows created by the tag join.

Before:

```
Crown Heights Anthem + rap
Crown Heights Anthem + hip-hop
Crown Heights Anthem + boom bap
```

After:

```
Crown Heights Anthem
```

The API now returns one result per song record.

---

### Side-effect checks performed

1. Verified multi-tag songs:

Request:

```http
GET /songs/search?q=rap
```

Checked that:

```
Crown Heights Anthem
Harlem Renaissance
```

appear only once.

---

2. Verified single-tag songs:

Example:

```python
("Free Throws", "Hoop Dreams", "rap", ["rap"])
```

Request:

```http
GET /songs/search?q=Free
```

Confirmed the song still appears correctly.

---

3. Verified songs without tags:

Example:

```python
("Midnight Drive", "The Wanderers", "indie rock")
```

Confirmed searching by title or artist still returns songs that do not have tag relationships.

---

4. Verified unrelated song functionality:

Checked:

```http
GET /songs/<song_id>
POST /songs/<song_id>/rate
POST /songs/<song_id>/listen
```

Confirmed:
- song retrieval still works
- ratings still work
- listening events still work

---

### Final result

The search endpoint now returns unique songs while still supporting tag-based searches.

The bug was caused by a many-to-many `song_tags` join producing duplicate rows for songs with multiple tags, and the fix ensures each song appears only once in search results.

---

## Issue #4 — Missing rating notification (`notification_service.py`)

### How I reproduced it

1. Started the Flask application.

2. Used Postman to rate an existing song:

```http
POST /songs/<song_id>/rate
```

Request body:

```json
{
  "user_id": "b82480fa-ba37-415b-b5cb-23cdabc4abb6",
  "score": 5
}
```

3. Confirmed that the rating request completed successfully and the rating record was created.

4. Checked notifications for the original song owner:

```http
GET /users/<song_owner_id>/notifications
```

5. Observed that no notification was created after the rating action.

### Actual Behavior

- Song rating was saved successfully.
- The song owner did not receive any notification.

### Expected Behavior

When a user rates another user's shared song:

- The rating should be stored.
- The original song owner should receive a notification informing them that their song was rated.

---

## How I found the root cause

### Navigation path

I traced the request flow from the API endpoint:

```
routes/songs.py
        |
        v  POST /songs/<song_id>/rate
        |
        v
services/notification_service.py
        |
        v
rate_song()
```

I then compared the rating flow with the playlist addition flow:

```
routes/playlists.py
        |   POST /playlists/<playlist_id>/songs
        v
services/notification_service.py
        |
        v
add_to_playlist()
```

The playlist flow created notifications using:

```python
create_notification(...)
```

but the rating flow did not have the same step.

### Moment I identified the root cause

While reviewing `rate_song()` in `notification_service.py`, I found that the function only handled:

- validating the score
- finding the song
- finding the user
- creating/updating the Rating record
- committing the database change

There was no call to:

```python
create_notification(...)
```

after a rating was created.

This confirmed that the missing notification was caused by missing notification logic, not by the notification retrieval API.

---

## Root Cause

The `rate_song()` function successfully stored ratings but did not trigger the notification side effect.

The playlist feature already created notifications after user actions, but the rating feature was missing this step.

The missing behavior was:

```
Rating created
        |
        X
Notification not created
```

Because `create_notification()` was never called, the song owner had no notification record to retrieve.

---

## Fix and Side-Effect Check

### Fix

Added notification creation after saving a new rating:

```python
create_notification(
    user_id=song.shared_by,
    notification_type="song_rated",
    body=f"{rater.username} rated your song '{song.title}'."
)
```

### Why this fixes the issue

The rating workflow now follows the same pattern as other user interactions:

```
User rates song
        |
        v
Rating saved
        |
        v
Notification created
        |
        v
Owner sees notification
```

---

## Side-Effect Checks

After applying the fix, I verified:

### 1. Rating functionality still works

Checked:
```http
POST /songs/<song_id>/rate
```

Result:

- Rating record created successfully.
- Existing rating updates still work.

---

### 2. Notification retrieval still works

Checked:

```http
GET /users/<song_owner_id>/notifications
```

Result:

- New rating notification appears correctly.

---

### 3. Existing playlist notification behavior was not affected

Verified:

```http
POST /playlists/<playlist_id>/songs
```

Result:

- Playlist addition notifications continue to be created.

---

### Final Result

The bug was caused by a missing service call, not by database or API retrieval issues.

Adding the notification creation step restored the expected behavior:

```
Song rated
    |
    v
Rating saved
    |
    v
Song owner notified
```

---

## Issue #5 — Last song missing in playlist (`playlist_service.py`)

### How to reproduce
```http
GET /playlists/<playlist_id>/songs
```

### Actual behavior
Returns only first 4 songs:
```
["Track 1", "Track 2", "Track 3", "Track 4"]
```

### Expected behavior
All songs returned in order:
```
["Track 1", "Track 2", "Track 3", "Track 4", "Track 5"]
```

### Root cause
Incorrect slicing:
```python
songs[:-1]
```

### Fix
Removed slicing, return full result set.
```python
songs[:]
```

### Result
✔ Full playlist returned correctly

---

## Summary
All issues were traced from service layer logic, primarily:
- incorrect filtering
- misuse of joins
- missing service calls
- incorrect slicing
- date boundary logic errors
