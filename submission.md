# AI Usage
I'm running out of credits for other stuff so I didn't use AI for this activity at all.
# /routes:
- routes/feed.py:
    + /listening-now: returns what the user's friends are currently listening
    + /activity: returns the most recent events, activities by the user's friends.
- routes/playlists.py:
    + /: create a new playlist. Request contains name, created_by, and optional field is_collaborative
    + /{playlist_id}: returns the playlist metadata, not including the songs
    + GET /{playlist_id}/songs: get the songs in the playlist
    + POST /{playlist_id}/songs: add a new song to the playlist. Request contains song_id and added_by
- routes/songs.py:
    + /search: search a song with a given query via "?q={query}"
    + /{song_id}: get a song metadata by id
    + POST /{song_id}/rate: add a rating to song with song_id. Request contains user_id and score
    + POST /{song_id}/listen: record that the user listen to the song with song_id and update the streak. Request contains user_id.
- routes/users.py:
    + /{user_id}: Get user's data
    + /{user_id}/streak: get user's current streak
    + /{user_id}/notifications: Request gives unread_only. Get the notifications for the user
    + POST /notifications/{notification_id}/read: mark the notification with notification_id as read

# /services
- services/feed_service.py: 
The logic for showing activity feed and what friends are listening to. Contains 2 functions get_friends_listening_now() and get_activity_feed(). The only difference between these 2 is that get_friends_listening_now() has a cutoff value
- services/notification_service.py:
Logic for creating notification, adding song to a playlist and notify the sharer, getting unread or read notifications of a user, and marking a notification as read. It also handles the song rating logic.
- services/playlist_service.py: playlists retrieval logic, including getting a user's playlists, a playlist's songs, a playlist's metadata, and creating a new playlist
- services/search_service.py: song search logic. search a song with a given name, or look up given song_id
- services/streak_service.py: listening streak logic. it records when a user listen to a song, updates the streak correspondingly, and also gets a user's streak.

# models.py
models.py defines 7 SQLAlchemy models: User, Tag, Song, ListeningEvent, Rating, Playlist, and Notification, plus 3 many-to-many association tables (friendships, song_tags, playlist_entries).

- User: id, username (unique), email (unique), listening_streak, last_listened_at, created_at.
    + Relationships: shared_songs (Songs they shared), ratings, listening_events, notifications, playlists (created), friends (symmetric self-referential many-to-many via friendships table)
- Tag: id, name (unique). A label attached to songs (e.g. "rap", "lo-fi") via the song_tags association table.
- Song: id, title, artist, album, genre, shared_by (FK -> user.id), shared_at, share_note.
    + Relationships: ratings, listening_events, tags (many-to-many via song_tags)
- ListeningEvent: id, user_id (FK -> user.id), song_id (FK -> song.id), listened_at. Records a single instance of a user listening to a song; used to derive "listening now"/activity feed and streaks.
- Rating: id, user_id (FK -> user.id), song_id (FK -> song.id), score (1-5), rated_at. Unique constraint on (user_id, song_id) so a user can only rate a song once.
- Playlist: id, name, created_by (FK -> user.id), created_at, is_collaborative.
    + Relationships: songs (many-to-many via playlist_entries, which also tracks position, added_by, and added_at for each song in the playlist)
- Notification: id, user_id (FK -> user.id), notification_type, body, created_at, read. Used to notify a user of events like a friend adding their shared song to a playlist.

Association tables:
- friendships: user_id, friend_id (both FK -> user.id, composite PK). Rows are inserted symmetrically (both directions) to represent bidirectional friendship.
- song_tags: song_id (FK -> song.id), tag_id (FK -> tag.id), composite PK. Many-to-many link between songs and tags.
- playlist_entries: playlist_id (FK -> playlist.id), song_id (FK -> song.id), composite PK, plus position, added_by (FK -> user.id), added_at. Many-to-many link between playlists and songs that also orders songs and tracks who added each one.

# Data flow
User listens to a song => POST to /{song_id}/listen -->
record_listening_event(user_id, song_id) is called -->
A new ListeningEvent is created, and added to the listening_event table -->
Then, the user's streak is updated with update_listening_streak.

# Issues fixed
## Issue 1: My listening streak keeps resetting
```
I found and reproduce it using the tests/ folder. That bug made the application failed both of the tests in test_streaks.py.
Next, I traced the actual tests and found out that the streak didn't increaseon Sunday, so I navigated to update_listening_streak() in streak_service
The cause was an elif clause that checked if today was not Sunday before increasing the streak. I simply deleted the conditional statement.
```
## Issue 2: Friends Listening Now shows people from yesterday
```
After reading the codebase, I know the root cause lies in get_friends_listening_now() of feed_service, so I navigate there directly.
The issue of showing people from yesterday ties clearly to the "cutoff" variable. Indeed, the math logic for comparing against "cutoff" is wrong:
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
datetime.now(timezone.utc) - ListeningEvent.listened_at < RECENT_THRESHOLD
=> ListeningEvent.listened_at > cutoff,
NOT ListeningEvent.listened_at >= cutoff
I verified it didn't change anything else by test running /feed/<user_id>/activity
```
## Issue 3: The same song keeps showing up twice in search
```
I think there is no issue with this. I tried multiple tests with:
GET http://127.0.0.1:5000/songs/search?q=...
and none of them returned duplicate results.
The code logic looks correct to me, also.
```
## Issue 4: I got notified when a friend added my song to a playlist but not when they rated it
```
I reproduced the issue with some POST request, for e.g:
POST http://127.0.0.1:5000/songs/c74c40b6-30e6-48b3-a002-c666bd5cb017/rate
Content-Type: application/json

{
  "user_id": "b042dda8-31f6-4dcd-8097-797b937885ba",
  "score": 3
}

This obviously lied in the rating logic, so I navigated to rate_song() of notification_service.py. I found out that the function only records the user's rating, but doesn't notify the sharer, so I added it with the same logic as add_song_to_playlist()
I verified it works by running the same POST commands again and check the notification tables, looking for the row that contains the sharer id.
```
## Issue 5: The last song in a playlist never shows up
```
I found and reproduce it using the tests/ folder, in particular test_playlist. For the test failed, they all showed the last song missing.
The corresponding endpoint is /<playlist_id>/songs, and the corresponding function is get_playlist_songs(). The return statement returns all the songs but the last one, so I simply fixed that.
The function is not used anywhere else so nothing else is affected.
```