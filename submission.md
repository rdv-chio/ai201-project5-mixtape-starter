# Codebase Map

## Architectural Design & System Patterns

Mixtape follows a strict, decoupled Route-Service-Model architecture. This separation of concerns ensures that the application layers handle distinct steps of the execution flow:

* Routing Layer (`routes/`): Acts as the interface gatekeeper. These files handle HTTP request parsing, URL parameter extraction, query argument handling, and JSON response serialization. They contain zero business logic and delegate instantly to the underlying services.

* Business Logic Layer (`services/`): The engine room of the application. These modules process data inputs, enforce specific business constraints (such as calculating streaks, managing feeds, or running text matching), and coordinate database transactions. All open bugs reside here.

* Data Access Layer (`models.py`, `app.py`): Defines the relational schema using `SQLAlchemy`. It explicitly establishes core tables, symmetric many-to-many relationships (friendships, playlist entries, and song tags), and handles application context/database initialization.

## Component & File Responsibilities

### Core Configuration & Schema

* `app.py`: The application factory. Dynamically provisions the Flask context, registers configurations, binds blueprints to explicit URL prefixes, and configures the `SQLAlchemy` engine state.

* `models.py`: The central data registry. Implements 5 relational mapping models (`User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Notification`) and 3 crucial junction association tables (`friendships` for symmetric social mapping, `song_tags` for metadata sorting, and `playlist_entries` which maintains explicit, sequential positioning of songs in playlists).

### Routing Layer (`routes/`)

* `routes/songs.py`: Exposes endpoints for text search querying, single song object retrieval, rating submissions, and posting play events.

* `routes/playlists.py`: Manages collaborative playlist lifecycle actions, including creating new entities, rendering song collections, and appending newly selected track items.

* `routes/users.py`: Services user profile serialization, live streak validation queries, and notifications status modification (marking alerts as read).

* `routes/feed.py`: Exposes routes for rendering temporal peer listening dashboards ("Listening Now") alongside generalized timeline records ("Activity").

### Service Layer (`services/`)

* `services/streak_service.py`: Evaluates temporal listening history to maintain consecutive calendar day tracking states for active accounts.

* `services/feed_service.py`: Aggregates, tracks, and filters database play events from verified friends to generate real-time social dashboards.

* `services/search_service.py`: Implements basic relational text query filters matching user input parameters against song titles and artist fields.

* `services/notification_service.py`: Processes transactional user interactions—such as playlist additions or track evaluations—and creates downstream notification items.

* `services/playlist_service.py`: Validates playlist permissions and queries positional junction assets to rebuild ordered track playlists.

## Detailed Data Flow Trace: Sharing & Interacting with a Song
To see how these layers cooperate, let's trace the exact data flow of a user adding a friend's shared song to a collaborative playlist:

```text
[Client Request] 
      │
      ▼
[routes/playlists.py: add_song()] 
      │  Captures parameters: playlist_id, song_id, added_by
      ▼
[services/notification_service.py: add_to_playlist()]
      │  1. Pulls Song, User (adder), and Playlist objects from db.session
      │  2. Appends Song object directly to playlist.songs collection
      │  3. Compares song.shared_by against added_by_user_id
      │  4. If mismatch occurs, triggers create_notification()
      ▼
[services/notification_service.py: create_notification()]
      │  1. Instantiates a new Notification db model
      │  2. Appends object to db.session and calls db.session.commit()
      ▼
[Database Commit] ──> Persists changes & fires alerts to the song's original author.
```