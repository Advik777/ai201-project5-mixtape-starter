### Mixtape Project Submission - Advik

### Milestone 1: Codebase Map

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

## Root Cause Analyses
(To be completed as bugs are fixed)

## how you reproduced it

 - Bug 1 Listening streak keeps resetting:  I inspected the logic in update_listening_streak. I simulated a listening event where days_since_last == 1 (consecutive day) but the current day was a Sunday (weekday() == 6). The condition today.weekday() != 6 failed, causing the streak to hit the else block and reset to 1.

 - Bug 2 Friends Listening Now shows people from yesterday: I examined feed_service.py and found that RECENT_THRESHOLD is hardcoded to 24 hours. I verified this by checking the feed for a user whose friends had listening events from 10+ hours ago; they were still incorrectly categorized as "Listening Now."

 - Bug 3 The same song keeps showing up twice in search: I performed a search for a song known to have multiple tags. Because the query uses an outerjoin on the song_tags table without a .distinct() filter, the resulting list contained duplicate entries for the same song ID, matching the number of tags associated with that song.

  - Bug 4 Missing notification for rating: I compared rate_song to add_to_playlist in notification_service.py. While add_to_playlist explicitly calls create_notification, the rate_song function only commits the rating to the database and lacks any notification trigger logic.

  - Bug 5 Last song in a playlist never shows up:  I called get_playlist_songs for a playlist containing multiple tracks. I compared the database count (3 songs) to the API response (2 songs) and confirmed that the final song in the ordered list was consistently omitted due to the Python slice [:-1] in the return statement.