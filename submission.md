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

### How to reproduce
Call:
```
POST /songs/<song_id>/listen
```

Body:
```json
{
  "user_id": "<user_id>"
}
```

Run on consecutive days.

### Actual behavior
Streak sometimes stays `1 → 1` instead of incrementing.

### Expected behavior
- +1 if consecutive day
- reset if gap > 1 day

### Root cause
Incorrect weekday constraint blocked valid updates:
```python
today.weekday() != 6
```

### Fix
Removed weekday dependency and used day-difference logic only.

### Result
✔ Correct streak increments across days

---

## Issue #2 — Friends Listening Now shows old activity (`feed_service.py`)

### How to reproduce
```
GET /feed/<user_id>/listening-now
```

### Actual behavior
Shows activity older than 24 hours.

### Expected behavior
Only last 24 hours of activity.

### Root cause
Used rolling timestamp incorrectly instead of strict cutoff.

### Fix
Replaced with proper calendar-day cutoff filter.

### Result
✔ Correct real-time feed behavior

---

## Issue #3 — Duplicate songs missing in search results (`search_service.py`)

### How to reproduce
```
GET /songs/search?q=rap
```

### Actual behavior
Songs do NOT duplicate per tag join as expected.

### Expected behavior
Same song appears multiple times (once per tag match).

### Root cause
- SQLAlchemy `query(Song)` collapses results by primary key
- `outerjoin(song_tags)` was ineffective

### Fix
- Ensure query is driven by `song_tags`
- Allow duplicate rows when join-based filtering is required
- Optionally use `DISTINCT` depending on filter logic

### Result
✔ Correct duplicate tag-based search behavior

---

## Issue #4 — Missing rating notification (`notification_service.py`)

### How to reproduce
```
POST /songs/<song_id>/rate
```

Body:
```json
{
  "user_id": "b82480fa-ba37-415b-b5cb-23cdabc4abb6",
  "score": 5
}
```

### Actual behavior
No notification is created.

### Expected behavior
Song owner receives notification on rating.

### Root cause
`rate_song()` did not call notification logic.

### Fix
Added direct notification trigger after rating.

### Improvement suggestion
Event-driven architecture:
```
Rating created → Event emitted → Notification service listener
```

Or centralized API:
```
notification_service.notify("song_rated", ...)
```

### Result
✔ Notifications now trigger correctly

---

## Issue #5 — Last song missing in playlist (`playlist_service.py`)

### How to reproduce
```
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
