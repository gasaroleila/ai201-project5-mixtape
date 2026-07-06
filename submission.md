# MixTape Bug Hunt

## Main Files & Functions:


### streak_service:

* record_listenting_event: Creates a ListeningEvent instance to show a user listened to a song at that time. Updates their streak as well using update_listening_streak.
* update_listening_streak: uses user’s last_listen_time. If the last_listen time is yesterday streak+=1, if it’s not yesterday reset streak to 1, and if it’s their first time listening in the app set streak = 1.
get_streak: fetches user.listen_streak field.

### feed_service:
* get_friends_listening_now: get the user’s friend who have ListeningEvents in the past 24 hours. Function fetches each friends past 24 hours listening events and the most recent’s listened_at is returned.
* get_activity_feed: returns limit(N) last listening events of the friend’s of a user.

### notification_service:
* create_notification: creates a new notification instance for user with body and notification type.
* add_to_playlist: saves a song to a playlist and notifies the original song sharer about it. 
* rate_song: saves song rating from user by updating an existing rating(if user has already rated before) or adding a new rating.
* get_notifications and mark_as_read does as the names suggest, gets notifications of a user(can filter to only unread ones) also mark it as read.

### playlist_service:
* get_playlist_songs(): Should return all songs in a playlist ordered by position field.
* get_playlist(): gets playlist metadata without songs.
* get_user_playlists(): returns all playlists by a user as metadata.

### search_service:
* search_songs(): searches all songs by title or artist matching to search text.
* get_song(): searches song by id

## Data Flow

Relevant features were traced to understand how data flows:

### Listening Streak
"How is a user's streak works, what services affect the user's streak ?"

user's streak is handled entirely by streak_service so any issue on streak should be coming from here. 

When a user wants to listen to a song, the '/<song_id>/listen' calls
record_listening_event which then calls update_listening_streak() and update_listening_streak() handles updating a streak or creating a fresh streak. Streak rules:

* first listen: streak becomes 1
* listened again on the same day: no change
* listened yesterday: streak increments by 1
* skipped one or more days: streak resets to 1

### Friends Listening Now Feed
"What displays friends listening now on a user's feed, in what order do these people get to a user's feed?"

From the '/<user_id>/listening-now', get_friends_listening_now is called and inside the ordering of getting these friends to a user's feed is:

It looks at the user’s friends.
It finds their ListeningEvent rows from the last 24 hours. -> This is the point which needs to be verified that it's truly events from last 24 hours and not just all past events.
It sorts those events by listened_at descending, so newest listens come first.
If a friend has multiple recent listens, it keeps only that friend’s most recent one.
The final feed is returned in that newest-first order.

### How Results are Returned After Search
"If a user searches a song how are results returned to them from API call to result?"

When the client calls '/search' in songs.py, it hits the search handler, which reads the query string parameter q and then calls search_songs(query) from search_service.py.

From there, search_songs runs a database query against Song, matches title or artist with a case-insensitive partial search, and returns a list of song dictionaries. The route then wraps that list in JSON like this:

results: the list returned by search_songs
count: the number of matches

Should look into what happens after search results are returned from database.
