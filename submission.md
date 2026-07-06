### Mixtape Project Submission - Advik

## AI Usage Disclosure
I used Copilot as a technical reference and sounding board during this project.

- Codebase Orientation: I used AI to clarify a few specific call chains between the routes and services to ensure I understood the intended architecture before I started debugging.

 - Technical Reference: I used AI to look up specific library behaviors when I had a hypothesis. For example, I confirmed that Python's weekday() returns 6 for Sunday to verify the root cause of Bug 1, and I checked the syntax for .distinct() in SQLAlchemy to resolve the duplicate issues in Bug 3.


## Core Architecture
The Mixtape application is built using a modular Flask architecture, following the Controller-Service-Model pattern. This ensures that the social music logic is cleanly separated from HTTP handling and data persistence.

app.py: The central configuration point. It uses the Flask factory pattern to initialize the SQLAlchemy db instance and register blueprints for different functional areas like songs, playlists, users, and the activity feed.

models.py: The data foundation. It defines SQLAlchemy models for the entire app, using UUIDs for all primary keys to ensure global uniqueness. Notable models include:
 - User: Tracks profile info and the listening_streak.

 - Song: Stores track details and uses the song_tags association table for many-to-many categorization.

 - Playlist: Uses the playlist_entries table to manage songs, specifically including a position column to preserve custom track ordering.
 - Notification: Stores alerts for users when friends interact with their shared content.

routes/: This directory acts as the entry point for API calls. For example, routes/songs.py manages searching and song-specific actions. These files focus on request validation and response formatting.

services/: The "brain" of the application where all business logic resides.
streak_service.py: Calculates and updates user streaks based on the timing of ListeningEvent entries.

notification_service.py: Manages the creation of Notification records and handles the persistence of song ratings.

## Data Flow Trace: Rating a Song
To understand how data moves through the system, here is the trace for when a user rates a song:

 - Request: A client makes a POST request to /songs/<song_id>/rate with a JSON body containing the user_id and a score (1–5).

 - Route: The rate function in routes/songs.py extracts the data. It performs a basic check for required fields and then calls the service layer.

 - Service: The logic moves to services/notification_service.py inside the rate_song function.

 - Logic & Model: The service retrieves the Song and User from the database to verify they exist. It then queries the Rating model to see if this user has already rated this specific song.

 - Persistence: If a rating exists, the service updates the score; otherwise, it creates a new Rating instance. It then calls db.session.commit() to save the changes to the database.

 - Response: The route receives the Rating object, converts it to a dictionary using to_dict(), and returns a 201 Created JSON response.

## Architectural Patterns
 - Thin Controllers: The routes are intentionally kept "thin," delegating all complex logic (like streak calculations or notification triggers) to the service layer.

 - Service Encapsulation: Services are designed to be the single source of truth for business rules. For instance, routes/songs.py calls both search_service and notification_service to complete its tasks.

 - UUID Identifiers: Every model uses a string-based UUID as its primary key, ensuring that IDs are unique across the system without relying on auto-incrementing integers.


## How you reproduced it

 - Bug 1 Listening streak keeps resetting:  I inspected the logic in update_listening_streak. I simulated a listening event where days_since_last == 1 (consecutive day) but the current day was a Sunday (weekday() == 6). The condition today.weekday() != 6 failed, causing the streak to hit the else block and reset to 1.

 - Bug 2 Friends Listening Now shows people from yesterday: I examined feed_service.py and found that RECENT_THRESHOLD is hardcoded to 24 hours. I verified this by checking the feed for a user whose friends had listening events from 10+ hours ago; they were still incorrectly categorized as "Listening Now."

 - Bug 3 The same song keeps showing up twice in search: I performed a search for a song known to have multiple tags. Because the query uses an outerjoin on the song_tags table without a .distinct() filter, the resulting list contained duplicate entries for the same song ID, matching the number of tags associated with that song.

  - Bug 4 Missing notification for rating: I compared rate_song to add_to_playlist in notification_service.py. While add_to_playlist explicitly calls create_notification, the rate_song function only commits the rating to the database and lacks any notification trigger logic.

  - Bug 5 Last song in a playlist never shows up:  I called get_playlist_songs for a playlist containing multiple tracks. I compared the database count (3 songs) to the API response (2 songs) and confirmed that the final song in the ordered list was consistently omitted due to the Python slice [:-1] in the return statement.

## Root Cause Analyses

## Bug 1: My listening streak keeps resetting
- How I reproduced it: I inspected update_listening_streak and identified a condition today.weekday() != 6 that explicitly blocked increments on Sundays (day 6 in Python's datetime). I simulated a consecutive day listen where the current day was Sunday, and confirmed the streak reset to 1.
- How I found the root cause: I traced the record_listening_event call to the update_listening_streak logic. The presence of a weekday check in a simple increment block was immediately suspicious.
- The root cause: The logic used today.weekday() != 6 as a guard clause for incrementing the streak. In Python’s datetime module, weekday() returns 6 for Sunday. This meant every Sunday, the condition would fail, skipping the increment and falling into the else block which resets the streak to 1.
- My fix and side-effect check: I removed the and today.weekday() != 6 condition. Now, as long as days_since_last == 1, the streak increments regardless of the day of the week. I ran test_streaks.py to ensure Monday-Saturday increments still work correctly.

## Bug 5: The last song in a playlist never shows up
- How I reproduced it: I requested the songs for a playlist known to have 3 entries via the API. The response consistently only returned 2 songs, omitting the final one.
- How I found the root cause: I navigated to services/playlist_service.py and looked at the return statement of get_playlist_songs. The use of Python slicing on the result list was the obvious culprit.
- The root cause: The function was returning songs[:-1]. In Python, the :-1 slice returns all items *except* the last one. This was a classic off-by-one error in the presentation layer of the service.
- My fix and side-effect check: I removed the [:-1] slice to return the full songs list. I verified that the order remains correct (ascending by position) and that all songs are now present in the output.

## Bug 2: Friends Listening Now shows people from yesterday
- How I reproduced it: I checked the "Listening Now" feed for a user whose friend had listened to a song 10 hours ago. The friend was still appearing in the feed despite the long inactivity.
- How I found the root cause: I looked at feed_service.py to see how "recent" was defined. I found the RECENT_THRESHOLD variable immediately.
- The root cause: The RECENT_THRESHOLD was set to timedelta(hours=24). This meant the "Listening Now" feature was actually a "Listened in the last 24 hours" feature, which is too broad for a real-time social feed.
- My fix and side-effect check: I changed the threshold to 1 hour. This ensures the feed only shows friends who are actually likely to be currently listening. I verified that friends with older events are now correctly excluded from this specific feed.

## Bug 3: The same song keeps showing up twice in search
- How I reproduced it: I searched for a song that I knew had multiple tags associated with it. The search results returned the same song multiple times—specifically, once for every tag the song possessed.
- How I found the root cause: I examined the query in search_service.py. I noticed it performed an outerjoin on the song_tags table. In SQL, joining a one-to-many relationship without grouping or distinct filtering results in duplicate parent rows for every child match.
- The root cause: The SQLAlchemy query lacked a .distinct() call. Because the outerjoin creates a result set row for every song-tag combination, a song with three tags produced three rows in the database result, which were then converted into three identical song objects in the final list.
- My fix and side-effect check: I added .distinct() to the query chain. This ensures that even if multiple rows are joined, only unique Song entities are returned. I verified this by re-running the search for multi-tag songs and confirmed that each song now only appears once.

## Bug 4: I got notified when a friend added my song to a playlist but not when they rated it
- How I reproduced it: I used the API to rate a song shared by another user. I then checked the notifications for the song's owner and found that no notification had been generated for the rating event, whereas adding to a playlist worked correctly.
- How I found the root cause: I compared the add_to_playlist and rate_song functions in notification_service.py. I noticed a clear architectural discrepancy: add_to_playlist had a call to create_notification, but rate_song did not.
- The root cause: The rate_song function was missing the logic to trigger a notification. While it correctly updated the database with the new rating, it failed to follow the established app pattern of alerting the content's original sharer about the interaction.
- My fix and side-effect check: I implemented a call to create_notification within the rate_song function, using the same conditional check (song.shared_by != user_id) used in other services. This ensures consistency across the notification system. I verified that the logic correctly identifies the song owner and formats the notification body with the rater's username and score.





