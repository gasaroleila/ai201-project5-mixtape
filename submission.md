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

## Root Cause Analysis

### Issue 1: Sunday Only

### How the bug was reproduced
I simulated a normal streak by adding consecutive days to the current time using now = datetime.now(timezone.utc) + timedelta(days=2), then days=3, etc and creating a listening event on each day which resulted in a streak of 4 days(The first day of the streak was Wednesday Jul 08) when I reached on Sunday, I listened to a song again and the streak was reset to 1 when it should have added 1 to be streak: 5.

{
    "id": "8501342d-af91-4b2e-8b85-b5d3f15ca5f1",
    "listened_at": "2026-07-12T02:41:53.777205",
    "song_id": "02833a11-7d3a-48c7-8617-f24c508a198d",
    "user_id": "d5e785e0-ecbb-4019-a729-783e02992c17"
}

{
    "streak": 1,
    "user_id": "d5e785e0-ecbb-4019-a729-783e02992c17"
}

Problem: Streak resets on Sunday.

### How the root cause was found
The API concerned is /songs/<song-id>/listen, this creates a ListeningEvent that calls record_listening_event which then calls update_listening_streak in streak_service. The bug happens when a user maintains a streak until Sunday when it resets so it has to do with a condition that resets the streak on this day.

### The root cause
This condition  "elif days_since_last == 1 and today.weekday() != 6:" explicity checks for consecutive listening as long as its not on Sunday 7. When it's on Sunday the condition doesn't pass and on a Sunday the else block ends up being executed which resets user.listening_streak = 1

### The fix
Removign the check today.weekday() != 6 to allow incrementing the streak even on Sunday. Checked that streak resets when they skip a day and that it increments even on Sunday. 

### Issue 2: Friends Listening Now shows people from yesterday
### How it was reproduced
After running GET http://127.0.0.1:5000/feed/d5e785e0-ecbb-4019-a729-783e02992c17/listening-now

Found the last_listened_at on friend and listened_at to be different which shouldn't be right so flagged this as a potential bug that might be related to this

     {
            "friend": {
                "id": "17201e18-c05a-436b-9a29-b61eeeed6eb8",
                "last_listened_at": "2026-07-05T00:21:02.004376",
                "listening_streak": 3,
                "username": "darius"
            },
            "listened_at": "2026-07-06T00:11:02.004376",
            ...
        },
        {
            "friend": {
                "id": "e91a283a-0afd-492e-ac47-11a43137bc00",
                "last_listened_at": null,
                "listening_streak": 0,
                "username": "simone"
            },
            "listened_at": "2026-07-06T00:06:02.004376",

### How the root cause was found
Dug into the what /feed/<id>/listening-now calls. And it only fetches the recent listening events of a user's friends, but these are registered under the record_listening_event in streak_service. So the feed_services only fetches these friends's past 24 hour listening events, the fact that people from yesterday appear is related to how the friends' last_listened_at is updated. So navigated to streak_service>record_listening_event then update_listening_streak.

### Root cause
Checking the code found that indeed the last listened date on the user is not updated when the user listened to songs on the same day. (ie days_since == 0) so the fetched friends come with an old last_listened_at that wasn't updated to match the external listened_at. This is a bug especially because in feed_service get_listening_now only filters by listened_at so the fetch person and last song is correct its just that fi they have listened to multiple songs in one day(days_since=0) their last_listened_at(on User model) won't be updated at all.

### The fix
Added the user.last_listened_at = now line under the when days_since_last = 0 to also update the last time a user listened even if their streak isn't updated because they listened to more than one song on the same day. Now a user's last listened even matches the record of their last ListenEvent.
