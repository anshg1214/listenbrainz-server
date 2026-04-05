# Proposal Review — Gopal (Playlist Features)

---

## Overall Assessment

This is a technically strong proposal. The candidate has clearly read the codebase, identified
real bugs with specific file paths and line numbers, and made deliberate architecture decisions
with justifications (TEXT[] vs junction table, MB replica vs REST API, view-only sorting). The
personal motivation is specific and credible. The four features are well-scoped for 175 hours.

The main issues are not about understanding — they are specific technical contradictions between
the stated design goals and the implementation chosen.

---

## Strengths

- Identifies two concrete bugs in the existing search endpoint with exact file and line context
- The decision to use the MB replica DB instead of the rate-limited REST API is sophisticated
and correct
- View-only sorting rationale (avoiding race conditions in collaborative playlists) is solid
- TEXT[] with GIN index vs junction table trade-off is well-argued
- The collaborative constraint ("tags must be personal") is the right design insight
- Stretch goal is explicitly conditional and realistic
- Timeline is 175 hours, not 90 — appropriately scoped

---

## Issues

### 1. Critical: Tags contradict the stated design constraint

The proposal correctly identifies that "a tag I apply to a collaborative playlist must not
appear for my collaborator." But the implementation stores tags in a `TEXT[]` column on
`playlist.playlist` — a row shared by all collaborators. If user A adds tag "gym" to a
collaborative playlist, user B sees "gym" too. This directly violates the stated constraint.

The folder schema gets this right — `playlist_folder` has a `creator_id` column, making folders
per-user. Tags need the same treatment: a separate `playlist_user_tag(user_id, playlist_id, tag)` table, not a column on the shared playlist row.

This is the most important issue in the proposal. The candidate should explain how they
reconcile the TEXT[] approach with their own design constraint.

### 2. The bug description may be inaccurate

The proposal says: "Bug 1 — search_playlists_for_user() passes include_global=True by default."

The actual function signature in `listenbrainz/db/playlist.py` is:

```python
def search_playlists_for_user(..., include_global: bool = False)
```

The default is already `False`. More importantly, the existing `/1/playlist/search` endpoint
does not call `search_playlists_for_user` at all — it calls a separate `search_playlist()`
function that only searches public playlists platform-wide. The proposed fix is a new
user-scoped endpoint, not a bug fix to an existing one. The candidate should clarify whether
they are fixing a bug or adding a new endpoint, and update the description accordingly.

### 3. Backend sort by additional_metadata fields may not work

Feature 3's backend sort uses:

```python
"title":  "lower(pr.additional_metadata->>'title') ASC",
"artist": "lower(pr.additional_metadata->>'artist_name') ASC",
```

The `playlist_recording` table's `additional_metadata` column is a JSONB field, but track
title and artist name are not stored there. They come from the MB recording lookup at read
time and are populated in the JSPF extension fields, not persisted in `additional_metadata`.
If these fields are null in the DB, the sort will silently put everything in the same order.
The candidate needs to explain where `title` and `artist_name` actually live in the DB row,
or propose how they will be populated before sorting.

### 4. MB DB connection style mismatch

The proposal uses `mb_engine.connect()` (SQLAlchemy style):

```python
with mb_engine.connect() as mb_conn:
```

But the actual codebase uses raw psycopg2:

```python
with psycopg2.connect(current_app.config["MB_DATABASE_URI"]) as mb_conn:
```

This is a minor inconsistency but suggests the code was written without verifying the existing
pattern in `entity_pages.py`.

### 5. MB OAuth for MusicBrainz not confirmed

The proposal says "The MB OAuth token is already stored in external_service_oauth under
service='musicbrainz'." The `external_service_oauth` table is used for Spotify, Apple Music,
and SoundCloud. Whether MusicBrainz OAuth is stored there, and under what service key, is not
confirmed in the codebase. This needs verification before Week 9.

### 6. Frontend calls MB REST API directly with OAuth token

For the collections list page, the proposal says "The collection list page calls the MB REST
API directly from the browser." This exposes the user's MusicBrainz OAuth token to client-side
JavaScript. The proposal says this is safe because it's "a single small request" but the
security implication of token exposure is separate from request size.

---

## Clarification Questions

### Feature 1 — Playlist Search

1. The existing `search_playlists_for_user()` already has `include_global=False` as the
  default. And the `/1/playlist/search` endpoint calls a different function entirely
   (`search_playlist`, not `search_playlists_for_user`). So what exactly is the bug you are
   fixing, and what exactly is new code? Can you describe the before/after in one sentence each?
2. When a search query is active and the user is on page 3 of their playlists, what happens?
  Does the search reset to page 1, or does it search within the current page? You mention
   "isolated component-level state" — how does this coexist with the URL-parameter pagination
   that the main list uses?
3. Your `searchPlaylistsForUser` in APIService.ts has a line:
  `const headers: HeadersInithorization: Token ${userToken} } : {};`
   This looks like a formatting error in the proposal. What is the complete correct line?

### Feature 2 — Tags

1. You stated: "organizational metadata must be personal to each user — a tag I apply to a
  collaborative playlist must not appear for my collaborator." But your implementation adds
   a `TEXT[]` column directly to `playlist.playlist` — a row shared by all collaborators.
   How does a column on a shared row give each user their own tags?
2. Your `get_playlists_by_tag` uses the `@>` containment operator, which means playlists must
  contain ALL the specified tags (AND logic). What happens if a user wants playlists tagged
   "chill" OR "gym"? Is OR filtering needed?
3. Tags are normalized to lowercase at the API layer. What happens if a user tries to rename
  a tag? Is there a rename endpoint, or do they delete the old tag and add the new one across
   all playlists manually?

### Feature 3 — Track Sorting

1. Your backend sort uses `pr.additional_metadata->>'title'` and
  `pr.additional_metadata->>'artist_name'`. Open `admin/timescale/create_tables.sql` and
   `listenbrainz/db/model/playlist.py` and tell me: are `title` and `artist_name` actually
   stored in the `additional_metadata` JSONB column on the `playlist_recording` row? If not,
   where do they come from, and how will your ORDER BY work?
2. You say `ReactSortable` accepts a `disabled` prop. Check the `react-sortablejs` source or
  docs and confirm this prop exists and disables dragging when true. What is the exact prop
   name and type?
3. Shuffle is handled client-side. If the user shuffles and navigates to page 2, the page 2
  order from the backend is position-based. How do you ensure a consistent shuffle order
   across pages?

### Feature 4 — MB Collections

1. You say "The MB OAuth token is already stored in external_service_oauth under
  service='musicbrainz'." Can you find where in the codebase this token is written to that
    table and confirm the service key string that is used?
2. The proposal uses `mb_engine.connect()`. Look at how `entity_pages.py` connects to the MB
  database. Is it using SQLAlchemy (`mb_engine`) or psycopg2 directly? Which pattern will
    your code follow?
3. The collection list page calls the MB REST API directly from the browser using the user's
  OAuth token. What are the implications of exposing an OAuth access token in client-side
    JavaScript? What happens if the token is intercepted?
4. Your SQL query for `mb_recording_to_jspf` has:
  `f"https://musicbrainz.org/recording/{recording.mbtion"`
    This is clearly cut off. What is the complete f-string, and what JSPF field does it map to?

---

## Follow-up Responses (Candidate Replies)

The candidate replied to five questions. Responses appear AI-generated (repetitive phrasing,
over-explained simple concepts, inconsistencies with the original proposal). Assessment below.

---

### Q1 — Search pagination coexistence

**Verdict: Conceptually right, poorly explained.**

The core answer is correct — search uses its own component-level page counter, URL params are
untouched, clearing search restores URL-based state. That's a sound implementation.

The explanation says "Search and the website address are two systems that do not change each
other" three separate times in slightly different words. No candidate who understood this
would need eight sentences to explain it. Ask them to re-explain in their own words.

---

### Q2 — Tags on shared row + rename

**Verdict: Correct fix. One schema bug.**

Acknowledging the flaw and switching to a junction table is the right call:

```sql
CREATE TABLE playlist_tag (
    playlist_id  UUID NOT NULL REFERENCES playlist.playlist(id) ON DELETE CASCADE,
    user_id      INT  NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    tag          TEXT NOT NULL,
    PRIMARY KEY  (playlist_id, user_id, tag)
);
```

The rename via `UPDATE playlist_tag SET tag = :new_tag WHERE user_id = :user_id AND tag =
:old_tag` is clean and correct.

**Bug:** `playlist_id UUID NOT NULL REFERENCES playlist.playlist(id)` — the primary key of
`playlist.playlist` is `id INTEGER` (SERIAL), not `mbid UUID`. The FK column type must be
`INTEGER`, not `UUID`. A candidate who opened the schema while writing this would not make
this mistake.

---

### Q3 — AND vs OR tag filtering

**Verdict: Correct.**

Both implementations are technically sound. `HAVING COUNT(DISTINCT tag) = array_length(:tags,
1)` for AND logic is correct. `match=all` / `match=any` as a query parameter is clean API
design. No issues.

---

### Q4 — title/artist_name in additional_metadata

**Verdict: Correct diagnosis. Incomplete solution.**

Correctly identifies that title and artist come from `get_playlist_recordings_metadata` via
MB lookup, not the DB row. In-memory sort after loading metadata is the right approach.

**Gap:** For a paginated playlist (500 tracks, 10 pages), you can't sort by title without
loading all 500 tracks' metadata first. The candidate says "We will use pagination when we
load sorted lists" without explaining how. If you're loading all tracks to sort them, what
does pagination mean? Ask for one more sentence of explanation on this.

---

### Q5 — Consistent shuffle across pages

**Verdict: Correct fix. Contradicts the original proposal.**

`ORDER BY md5(pr.mbid::text || :seed)` is the right pattern for deterministic server-side
shuffle. Seed in React state, passed as a query param on each paginated fetch, is clean.

**Problem:** The original proposal explicitly said "Shuffle is handled client-side only — no
backend state needed." This answer replaces that with a server-side sort. The proposal needs
to be updated — the sort handling section now has two incompatible designs.

---

### Summary

| Question | Answer quality | Action needed |
|----------|---------------|---------------|
| Q1 — Pagination | Conceptually right, AI-generated explanation | Re-explain in own words |
| Q2 — Tags junction table | Correct fix, one FK type bug | Fix `playlist_id UUID` → `INTEGER` |
| Q3 — AND/OR | Correct | None |
| Q4 — Sort metadata | Right diagnosis, pagination gap | Explain how pagination works with full-load sort |
| Q5 — Shuffle seed | Correct fix | Update proposal — shuffle is now server-side, not client-side |

---

## Timeline Questions

1. Feature 3 (Track Sorting) is in Week 11 at 12 hours. It involves both a backend sort
  parameter added to the existing playlist endpoint and a frontend sort dropdown. Is 12 hours
    enough if the `additional_metadata` sort fields need to be rethought?
2. Week 8 is "Folders frontend — drag-and-drop reordering" at 12 hours. Drag-and-drop for
  folder ordering is additional complexity on top of the existing ReactSortable drag-and-drop
    for tracks. Have you accounted for the interaction between the two drag systems on the same
    page?

