# Mixtape Bug Hunt 

## Codebase Map

### Architectural Design & System Patterns

Mixtape follows a strict, decoupled Route-Service-Model architecture. This separation of concerns ensures that the application layers handle distinct steps of the execution flow:

* Routing Layer (`routes/`): Acts as the interface gatekeeper. These files handle HTTP request parsing, URL parameter extraction, query argument handling, and JSON response serialization. They contain zero business logic and delegate instantly to the underlying services.

* Business Logic Layer (`services/`): The engine room of the application. These modules process data inputs, enforce specific business constraints (such as calculating streaks, managing feeds, or running text matching), and coordinate database transactions. All open bugs reside here.

* Data Access Layer (`models.py`, `app.py`): Defines the relational schema using `SQLAlchemy`. It explicitly establishes core tables, symmetric many-to-many relationships (friendships, playlist entries, and song tags), and handles application context/database initialization.

### Component & File Responsibilities

#### Core Configuration & Schema

* `app.py`: The application factory. Dynamically provisions the Flask context, registers configurations, binds blueprints to explicit URL prefixes, and configures the `SQLAlchemy` engine state.

* `models.py`: The central data registry. Implements 5 relational mapping models (`User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Notification`) and 3 crucial junction association tables (`friendships` for symmetric social mapping, `song_tags` for metadata sorting, and `playlist_entries` which maintains explicit, sequential positioning of songs in playlists).

#### Routing Layer (`routes/`)

* `routes/songs.py`: Exposes endpoints for text search querying, single song object retrieval, rating submissions, and posting play events.

* `routes/playlists.py`: Manages collaborative playlist lifecycle actions, including creating new entities, rendering song collections, and appending newly selected track items.

* `routes/users.py`: Services user profile serialization, live streak validation queries, and notifications status modification (marking alerts as read).

* `routes/feed.py`: Exposes routes for rendering temporal peer listening dashboards ("Listening Now") alongside generalized timeline records ("Activity").

#### Service Layer (`services/`)

* `services/streak_service.py`: Evaluates temporal listening history to maintain consecutive calendar day tracking states for active accounts.

* `services/feed_service.py`: Aggregates, tracks, and filters database play events from verified friends to generate real-time social dashboards.

* `services/search_service.py`: Implements basic relational text query filters matching user input parameters against song titles and artist fields.

* `services/notification_service.py`: Processes transactional user interactions—such as playlist additions or track evaluations—and creates downstream notification items.

* `services/playlist_service.py`: Validates playlist permissions and queries positional junction assets to rebuild ordered track playlists.

### Detailed Data Flow Trace: Sharing & Interacting with a Song
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

## Exhaustive Bug Hunt Report

### Issue #1: My listening streak keeps resetting

* **How you reproduced it**: Ran `pytest tests/` locally, revealing an explicit failure at `tests/test_streaks.py::test_streak_increments_on_sunday`. The test attempted to step an active streak from Saturday (`2024-06-15`) into Sunday (`2024-06-16`), resulting in an unexpected reset back to `1` instead of properly incrementing to `2`.

* **How you found the root cause**: Opened `services/streak_service.py` and traced the execution down to the `update_listening_streak(user, now)` logic engine. Line 77 showed a conditional constraint: `elif days_since_last == 1 and today.weekday() != 6:`.

* **The root cause**: Python's native `datetime.weekday()` maps the day sequence starting from Monday as `0` through Sunday as `6`. By incorporating `and today.weekday() != 6`, the application explicitly blacklisted Sunday as a valid day for sequential streak progression. Any listening event occurring on a Sunday dropped past the increment block and fell into the `else:` fallback clause, resetting the user's consecutive day streak down to 1.

* **Your fix and side-effect check**: Modified `services/streak_service.py` to remove the fragile and unnecessary weekday constraint. Because `days_since_last == 1` completely establishes structural day-to-day consecutiveness mathematically, the secondary check was completely cut out to allow clean transitions across all calendar days. Re-running `pytest tests/` verified that all 5 streak tests now pass flawlessly without breaking same-day deduplication logic.

### Issue #2: Friends Listening Now shows people from yesterday

* ****How you reproduced it****: Examined `seed_data.py` and noted that older historical events are seeded several hours to days into the past. Querying the live endpoint `/<user_id>/listening-now` returned friends whose last listening events occurred up to 23 hours ago, generating a cluttered social overview instead of showing who is streaming songs at this exact moment.

* **How you found the root cause**: Navigated to `services/feed_service.py` and investigated the configuration properties at the top of the module. Found that `RECENT_THRESHOLD` was explicitly configured with a massive historical window.

* **The root cause**: The business logic layer defined `RECENT_THRESHOLD = timedelta(hours=24)`. While a rolling 24-hour limit is optimal for a historical activity timeline, it creates a poor experience for a real-time "Listening Now" feature by treating stale events from yesterday as active live sessions.

* **Your fix and side-effect check**: Updated `RECENT_THRESHOLD` in `services/feed_service.py` to `timedelta(hours=1)` to represent an accurate, high-fidelity real-time window. Re-queried the system and cross-referenced with `routes/feed.py` to ensure that `get_activity_feed()` remains unaffected by this limit and still serves comprehensive historical records up to its designated 20-item capacity constraint.

### Issue #3: The same song keeps showing up twice in search

* **How you reproduced it**: Executed the internal test suite and captured a clear failure at `tests/test_search.py::test_search_no_duplicates_multi_tag_song`, where a search query for a song with multiple tags returned a result count of 3 instead of the expected single unique song dictionary mapping.

* **How you found the root cause**: Opened `services/search_service.py` and evaluated the `search_songs(query)` routine. Inspected the database retrieval call chain, noting the structural configuration of the outer join layout.

* **The root cause**: The SQLAlchemy query engine performed an unconstrained flat `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` on the many-to-many tag junction table. Because the query lacked an explicit distinct modifier before compiling results via `.all()`, the database returned an independent duplicate row for every single matching relational tag map link found in the bridge table.

* **Your fix and side-effect check**: Appended a targeted `.distinct()` call modifier immediately into the SQLAlchemy query builder chain right before the final `.all()` resolution. Re-ran `pytest tests/`, confirming that songs with zero, single, or 3+ associated tags return exactly one high-level distinct entity map while maintaining case-insensitive title and artist filters.

### Issue #4: Missing notification when a friend rates my song
**How you reproduced it**: Traced the endpoint interaction pattern in `routes/songs.py` under the `/rate` path. Submitting a POST payload smoothly persisted rating changes inside the database, but checking the corresponding recipient's alerts via `/<user_id>/notifications` confirmed that no transactional item was generated.

* **How you found the root cause**: Opened `services/notification_service.py` and performed a structural, line-by-line comparison between the fully working `add_to_playlist` action flow and the silent rate_song routine.

* **The root cause**: While `add_to_playlist` features a functional evaluation that calls `create_notification()`, the `rate_song` function was architecturally incomplete. The codebase processed the relational database updates for the score payload and executed `db.session.commit()`, but completely omitted any downstream pathway to dispatch an alert entry back to the song's original curator (`song.shared_by`).

* **Your fix and side-effect check**: Appended an intentional downstream alert condition inside `rate_song` directly following the database transaction. It verifies `if song.shared_by != user_id:` to prevent self-triggering alerts, and dispatches a cleanly constructed message via `create_notification()`. Validated that submitting new ratings now builds functional alerts for peer accounts without altering normal rating score calculation histories.

### Issue #5: The last song in a playlist never shows up

* **How you reproduced it**: Ran the testing engine, which halted on two systemic failures: `tests/test_playlists.py::test_playlist_returns_all_songs` and `tests/test_playlists.py::test_playlist_returns_songs_in_order`. The assertion logs confirmed that a playlist seeded with 5 sequential tracks systematically returned an array size of only 4.

* **How you found the root cause**: Opened `services/playlist_service.py` and inspected the close of the data extraction logic inside `get_playlist_songs(playlist_id)`.

* **The root cause**: The function concluded with an explicit array slicing instruction: `return [song.to_dict() for song in songs[:-1]]`. The choice of `[:-1]` syntax forced Python to evaluate elements from index zero up to, but explicitly excluding, the final item in the sequence. This introduced an intentional off-by-one error that dropped the trailing song from every playlist payload.

* **Your fix and side-effect check**: Removed the truncated slice syntax to cleanly return the complete list comprehension array: `return [song.to_dict() for song in songs]`. Re-ran `pytest tests/`, watching the final remaining failures pass instantly. Verified that empty playlists still safely return clean empty lists while fully populated playlists preserve their exact database insertion ordering.

## AI Usage Section

* **Codebase Orientation**: AI was used to parse the initial structure of the `services/` layer. By pasting the contents of complex junction queries, it helped map out how many-to-many relationships like `playlist_entries` and `song_tags` were handled by the underlying database engine.
* **Debugging Assistance**: AI was utilized to trace the specific behaviors of native Python time libraries. Specifically, it provided immediate clarity on how `datetime.weekday()` interfaces with calendar transitions.
* **Verification and Override**: While AI correctly hypothesized that `today.weekday() != 6` was a boundary failure, its initial structural suggestion was to replace it with complex ISO calendar lookups. This suggestion was overridden in favor of a cleaner, more readable solution: removing the fragile weekday checking wrapper entirely since the consecutive calculation was already fully handled by verifying `days_since_last == 1`.

```text
(ai201) rociodv@WIN-MU9C0LJD9CM:~/code/ai201-project5-mixtape-starter$ git log --oneline
9d93179 (HEAD -> bugfix/mixtape) Milestone 4: Final Review and AI Usage
531f6e3 (origin/bugfix/mixtape) fix: remove off-by-one array slice to prevent dropping last playlist song
542e80d fix: dispatch notification to song owner on new rating events
2eb6103 fix: enforce distinct rows in song search query to avoid duplicates
e4e4d67 fix: adjust rolling recency window for listening now feed
964e40b fix: correct Sunday boundary condition in streak reset logic
e306d0a (origin/main, origin/HEAD, main) Milestone 1: Fork, Set Up, and Orient Yourself
2dfdeaa Add .gitignore file and update README with setup instructions
7b64551 initial commit
(ai201) rociodv@WIN-MU9C0LJD9CM:~/code/ai201-project5-mixtape-starter$
```