# Proposal Review — FAILURE (MusicBrainz Events in ListenBrainz)

---

## Overall Assessment

This proposal shows genuine architectural understanding of ListenBrainz. The candidate correctly
identifies the two-database architecture (Postgres + TimescaleDB), reuses the existing
`user_relationship` pattern for artist follows, and adapts the `user_timeline_event` pattern for
notifications. The schema design is thoughtful and consistent with how similar caches already
work in the codebase. The honesty about weak sections (mockups, section 7) is better than
overconfident hand-waving with no acknowledgement.

The critical gap is section 7: the MB event cache ingestion pipeline is the hardest and most
time-consuming part of the entire project, and it is explicitly marked as incomplete. The
timeline has since been submitted and is analysed below.

---

## Strengths

- Correctly identifies the two-database split and explains how queries span both
- `user_artist_relationship` correctly mirrors `user_relationship` in structure and placement
- Reusing `user_timeline_event` for event notifications is exactly the right approach
- `fetch_track_metadata_for_items` pattern correctly identified and adapted
- Performer link_type_gid filtering is a subtle but important detail most proposals miss
- `event_interaction` table (watch/interested) is a thoughtful addition
- ~~Location area hierarchy approach (parent area UUID array) is architecturally sound~~ *(candidate dropped this in updated proposal — now uses MB API for area filtering)*
- Documentation-first mindset is genuinely rare and valuable

---

## Issues

### 1. Section 7 (cache ingestion) is the core of the project and is entirely missing

The proposal says "I will apologize preemptively for all the hand waving." But the MB event
cache ingestion is not a minor detail — it is the majority of the backend work. Without it:

- No events appear on artist pages
- No notifications trigger
- The Events Explorer has no data

The proposal needs to describe:

- How events are fetched from MB (MB DB replica via `MB_DATABASE_URI`, or MB REST API?)
- How often the cache refreshes (cron schedule)
- How the dirty flag is set and cleared
- The actual SQL that queries `musicbrainz.event`, `musicbrainz.l_artist_event`, and
`musicbrainz.link_type` in the MB replica

This is comparable in complexity to `mb_artist_metadata_cache.py` which the candidate
references. The proposal should describe the equivalent for events.

### 3. `make_event_date()` function does not exist

The DB query uses:

```python
make_event_date(ec.begin_date_year, ec.begin_date_month, ec.begin_date_day) BETWEEN %s AND %s
```

This function is not defined anywhere in the codebase. MB stores partial dates (year-only,
year-month, or full date). How does the query handle an event where only the year is known?
The candidate should define this function or explain the date comparison logic for partial
dates.

### 4. Python syntax bug in `get_watched_events`

```python
return results = db_conn.execute(query, (user_id, limit, offset)).fetchall()
```

This is invalid Python syntax. `return` and assignment cannot be combined on the same line
outside a walrus operator. This is a copy-paste artifact that should be caught before
submission.

### 5. Notification cron job will re-notify on every cache update

The cron job uses:

```python
new_events = get_events_ingested_since(ts_conn, last_run)
# basically last_updated > last_run
```

The `last_updated` column is updated every time the cache refresh runs (including when event
details like descriptions change). This will re-send notifications every time any event record
is touched, even for events the user was already notified about. The condition should
distinguish between first ingestion of an event and subsequent updates. A boolean
`notification_sent` flag on the event cache row, or a separate `notified_at` timestamp, would
prevent duplicate notifications.

### 6. Typos in code samples

- `save_last_run_timestanp` (missing 'i') in the cron job
- `metadata=medata` (missing 't') in `EventNotificationMetadata` constructor
- `#GSoC Application – MetaBrainz` appears in the middle of the DELETE query SQL string
(a copy-paste artifact)

### 7. Schema still has `area_id UUID[]` but Section 6 drops it *(new in updated proposal)*

Section 6 now explicitly removes the parent-area-array approach: "I don't like the above
solution... it's getting the axe." Area filtering will use the MB API instead. But the
`mb_event_cache` schema in Section 1.b still has `area_id UUID[]` as a column. If the
MB-API approach is final, this column should be removed from the schema definition. Leaving
it in causes confusion about what actually gets stored.

### 8. Area filtering via MB API reintroduces the rate limit problem *(new in updated proposal)*

The candidate correctly used the MB replica DB (not the REST API) for event data to avoid the
1 req/sec rate limit. But Section 6 now delegates area filtering back to the MB API
(`/ws/2/event?area=...`). The Events Explorer page may have many users filtering by area
simultaneously. What happens when the rate limit is hit?

### 9. `location_mbid` column target is contradictory

The proposal text says "add a column to store the location MBID in the `public.user` table"
but the SQL statement targets `user_setting`:

```sql
ALTER TABLE user_setting ADD COLUMN location_mbid UUID;
```

`user_setting` is the correct place (it already stores timezone and other preferences), but
the description should match the code.

### 9. `event_mbid` and `event_id` both in mb_event_artist_cache

The `mb_event_artist_cache` table stores both `event_mbid UUID` and `event_id INTEGER`. The
primary key on `mb_event_cache` is `event_mbid`, but the JOIN in the query uses `event_id`.
Storing both is redundant. The candidate should explain why both are needed, or simplify to
use only `event_mbid` as the FK with a matching index.

---

## Clarification Questions

### Schema

1. You use `make_event_date(...)` in your SQL query. This function doesn't exist in the
  codebase. How do you handle partial MB dates — for example, an event where only the year
   is known? What does `make_event_date(2026, NULL, NULL)` return, and how does the BETWEEN
   comparison work?
2. You dropped the `area_id UUID[]` parent-hierarchy approach and switched to using the MB
   REST API for area filtering. But your `mb_event_cache` schema still has `area_id UUID[]`
   as a column. Will you remove it? And if you are using the MB API for area filtering, what
   happens when the 1 req/sec rate limit is hit on the Events Explorer page?
3. Both `mb_event_cache` and `mb_event_artist_cache` store `event_id INTEGER`. The join uses
  `event_id`. Why not use `event_mbid` as the FK and drop the integer ID entirely from
   `mb_event_artist_cache`?

### Cache Ingestion (Section 7)

1. How does event data get from MusicBrainz into `mb_event_cache`? Is it via the MB replica
  database (`MB_DATABASE_URI`) or the MB REST API? Show the MB-side SQL query that fetches
   event rows.
2. How often does the ingestion cron run? Daily? Hourly? How do you decide which events to
  re-fetch (dirty flag mechanism)?
3. Your cron job uses `last_updated > last_run` to find new events and sends notifications.
  What happens when an event's description is updated in MB and the cache re-ingests it?
   Does the user get a duplicate notification?

### Notifications

1. Your cron job sends notifications to both `followers` (users who follow the artist) and
  `watchers` (users who watch the event). If a user both follows an artist AND watches the
   event, they get one notification or two? How does `deduplicate()` work across those two
   lists?
2. How far in advance does the notification trigger? If an event is six months away and gets
  added to MB today, does the user get notified immediately, or only when the event is within
   a certain window (e.g., 30 days)?

### General

1. Your code has `return results = db_conn.execute(...).fetchall()`. What should this line
  actually be?

### Codebase-Informed Questions

1. The existing `user_relationship` table uses `user_0` and `user_1` as column names. You used
   `user_id` in `user_artist_relationship`. Was this deliberate, or did you not notice the
   naming convention? If deliberate — why?

2. Your `mb_event_cache` has a `dirty BOOLEAN DEFAULT FALSE` column. In `mb_artist_metadata_cache`,
   dirty rows are set to `TRUE` by the MB change-detection process and cleared to `FALSE` after
   re-ingestion. What sets `dirty = TRUE` for events? Where does that code live in your design?

3. The codebase already has a `background_worker_state` table with
   `select_metadata_cache_timestamp()` / `update_metadata_cache_timestamp()` functions for
   exactly this purpose. Your cron uses a new `save_last_run_timestamp()` function. Why not
   use the existing state tracking mechanism?

4. You write `class MusicBrainzEventMetadataCache(MusicBrainzEntityMetadataCache)`. The base
   class requires implementing at minimum `get_create_table_columns()`, `get_index_names()`,
   and `create_json_data()`. What would your `create_json_data()` return — what fields go into
   the JSONB, and what stays as typed columns?

5. When you add `event_notification` to `user_timeline_event_type_enum`, there is a second enum
   that also needs updating: `hide_user_timeline_event_type_enum`. Should users be able to hide
   event notifications from their feed? Is this covered in your proposal?

6. `fetch_track_metadata_for_items` works because recordings have `recording_msid` /
   `recording_mbid`. Your event notifications only store `event_mbid`. What does your
   equivalent enrichment function look like — what does it query, on which connection
   (`db_conn` or `ts_conn`), and how does it attach the result to the timeline event before
   returning it to the frontend?

7. Your SQL `ALTER TYPE` adds `event_notification` to the database enum. There is also a Python
   `UserTimelineEventType(StrEnum)` in `listenbrainz/db/user_timeline_event.py` that needs a
   matching `EVENT_NOTIFICATION` value. Did you account for this, or only the SQL side?

### Nitpick — Cache Schema

1. Your index is named `user_id_user_arst_relationsip_ndx`. Read that name carefully — what
   is wrong with it?
2. `mb_event_cache` has a `dirty BOOLEAN DEFAULT FALSE` column. What sets it to `TRUE`, and
   what sets it back to `FALSE`? Where does that code live?
3. `event_art_presence` is defined as `TEXT NOT NULL DEFAULT 'absent'`. MB defines this as a
   fixed set of values: `absent`, `darkened`, `present`. Why `TEXT` instead of an enum?
4. `place_id UUID` — MB's `place` table has both an integer `id` and a UUID `gid`. Which one
   are you storing in this column?
5. `mb_event_artist_cache` stores both `event_mbid UUID` and `event_id INTEGER`, but your JOIN
   uses `event_id`. What does keeping `event_mbid` here give you that joining back to
   `mb_event_cache` on `event_id` wouldn't?
6. The primary key on `mb_event_artist_cache` is `(event_id, artist_id, link_id)`. Can one
   artist have two different relationship types to the same event — say, main performer and
   conductor simultaneously? If yes, does this primary key handle that correctly?
7. `area_id UUID[]` is still in your `mb_event_cache` schema. Section 6 says that approach is
   "getting the axe." Is this column in your final schema or not?

---

## Timeline Analysis

The submitted timeline targets the ~350-hour scope (25–30 hours/week × 12 weeks, minus 2 exam
weeks at ~10 hours). This is the right scope for the work proposed.

### What works well

- Front-loading the hardest work: the cache builder is Week 2, not Week 10. This is correct —
everything else depends on it.
- Exam schedule accounted for honestly: Weeks 3–4 are ~10 hours each, which maps to the exam
period (June 10–24). The work assigned to those weeks (Postgres schema and DB helpers) is
deliberately lightweight.
- Three documentation passes (Weeks 6, 9, 11) distributed throughout, not lumped at the end.
- Midterm deliverable is a working backend that can be demoed independently. This is a good
milestone target.
- Notification pipeline placed correctly in Week 10 — after the full stack is working.

### Timeline Issues

**Week 2 is the single most dangerous week in the proposal.**
The cache builder requires: querying the MB replica DB for events, joining across
`musicbrainz.event`, `musicbrainz.l_artist_event`, `musicbrainz.link_type`, and
`musicbrainz.place`; filtering by `link_type_gid`; handling partial dates (the `make_event_date`
problem); building the dirty flag mechanism; writing everything to two Timescale tables; and
wiring into the cron system. The candidate acknowledges this ("I won't sugarcoat it, this is
going to be a dense week") but no contingency plan exists if it overruns. If Week 2 slips, it
blocks Weeks 5, 6, 7, 8, 10, and 11.

**Week 6 title is truncated** — "Wee & First Docs" should be "Week 6 & First Docs". Minor but
worth fixing.

**Week 7 area filtering via MB API is now the stated approach** *(resolved in updated proposal)*
The candidate dropped the `area_id UUID[]` cache column and committed to the MB API for area
filtering. The contradiction is resolved. However, the `area_id UUID[]` column is still in
the Section 1.b schema definition and needs to be removed.

**Week 10 duplicate notification problem is not resolved.**
The timeline says "deduplication via last-run timestamp" which is the same approach flagged in
the main review. `last_updated > last_run` will re-notify on any cache refresh that touches the
event row, not just on first ingestion. The timeline does not address this.

**Week 12 is doing too much.**
End-to-end testing, edge case testing, final documentation, code cleanup, and the GSoC final
report are all in Week 12. If any bug is found during testing, there is no buffer to fix it.
Testing should begin in Week 11 at the latest.

**No mention of the Event detail page data sources.**
Week 8 builds the Event detail page with "links to tickets and homepage." Ticket purchase links
and venue homepages are not guaranteed to exist in MB event data. The timeline should note how
missing data is handled (graceful omission vs. placeholder).

---

## Additional Timeline Questions

1. Week 2 is the most complex week in the entire project. What is your contingency plan if the
  cache builder takes longer than one week? Which weeks can absorb the spillover?
2. You dropped the `area_id UUID[]` approach but your Section 1.b schema still shows that
   column. Will you remove it from the final schema? And given you are now using the MB API
   for area filtering, how do you handle the 1 req/sec rate limit if many users are browsing
   the Events Explorer simultaneously?
3. Week 10 says "deduplication via last-run timestamp." If an event's venue changes in MB
  and the cache re-ingests it, `last_updated` gets bumped. The user gets notified again.
    How do you prevent this?
4. Week 12 has end-to-end testing, edge case testing, documentation, code cleanup, and the
  final report. If testing uncovers a bug in the notification pipeline, when do you fix it?
5. Week 8 builds the Event detail page with "links to tickets and homepage." How does the
  Event page behave when MB has no ticket URL and no event homepage for that event?

