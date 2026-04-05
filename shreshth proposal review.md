# Proposal Review — Shreshth Sharma (Playlist Sorting and Organization)

---

## Overall Assessment

This is a well-structured proposal. The candidate has read the codebase carefully, correctly
identifies existing infrastructure to reuse (TagsComponent, AddTagSelect, is_modifiable_by,
JSPF pipeline), and makes deliberate architecture trade-offs with justifications. The execution
order (1 → 3 → 2 → 4) is strategically sound — smaller PRs early, complex migration in the
middle, independent feature last. The tag design is more carefully considered than most
proposals at this level.

The issues below are specific and technical, not gaps in understanding.

---

## Strengths

- Strategic PR ordering (Search → Sort → Tags → Collections) maximizes reviewable output early
- Correctly reuses AddTagSelect from TagsComponent.tsx rather than rebuilding from scratch
- Explicitly covers both AND (`@>`) and OR (`&&`) logic for tag filtering
- Acknowledges DOM virtualization limitation honestly and sets scope boundary clearly
- `mb_collection_prefs` table for hiding collections is a thoughtful addition
- References `is_modifiable_by()` and `is_visible_by()` by correct method names
- Identifies the read-modify-write race condition for tag updates and proposes a solution
- Note on caching (intentionally skipped for live-view goal) shows system-level thinking
- Timeline has a rolling buffer and realistic hour counts

---

## Issues

### 1. "Release Date" sort has no data to sort on

Sub-project 3 includes "Release Date" as a sort option in the dropdown. The JSPF track object
in ListenBrainz contains: `title`, `creator` (artist), `album` (release name), `duration`,
and `identifier`. There is no release date field. A sort by release date would silently put
all tracks in the same order. Either this option should be removed, or the proposal should
explain how the release date would be fetched and where it would be stored.

### 2. Tag rename is in the UI but missing from the endpoints

The Tag Manager Modal mockup (gear icon) explicitly supports "create, rename, or delete" tags
globally. But the API section only defines:
- `POST /1/playlist/<mbid>/tags` (add)
- `DELETE /1/playlist/<mbid>/tags/<tag>` (remove one tag)

Renaming a tag across all of a user's playlists is a different operation — it requires
finding every playlist where that tag exists and replacing it. There is no endpoint for this.
The candidate should either describe the rename endpoint, or clarify that rename is
implemented as delete-old + add-new across each playlist manually.

### 3. Tags are "personal organization" but stored at the playlist level

The proposal says tags are for "personal organization" and that "LB tags are for personal
organization." But the `TEXT[]` column is on `playlist.playlist`, a row shared by all
collaborators. If user A (collaborator) tags a playlist "gym", user B (the creator) also sees
"gym" when they view their own playlists.

This is less of a flaw and more of a design clarification needed. Shreshth's design is
different from Gopal's — here, tags are playlist-level labels that collaborators share, not
private-per-user metadata. That's a valid design. But calling them "personal organization"
is misleading if all collaborators see the same tags. The candidate should clarify: are tags
meant to be shared among collaborators, or truly private to each user?

### 4. MB Collections requires `musicbrainz_row_id`, not `musicbrainz_id`

To query a user's collections from the MB replica, the backend needs the MB editor's integer
ID. The `editor_collection` table has `editor INTEGER` which maps to `editor.id` in MB — not
the username string.

In the LB `user` table, the column `musicbrainz_row_id` stores exactly this integer (the MB
editor's primary key). The proposal does not mention this lookup, which means the endpoint
`GET /1/user/<user_name>/musicbrainz-collections` would need to:
1. Look up the LB user by username
2. Use `user.musicbrainz_row_id` as the `editor` value in the MB query

The candidate should confirm they are aware of this and describe where this lookup happens.

### 5. `SELECT ... FOR UPDATE` is more complex than necessary for array operations

For the tag update race condition, the proposal uses a read-modify-write cycle wrapped in
`SELECT ... FOR UPDATE`. PostgreSQL's native array functions (`array_append`, `array_remove`)
are single-statement atomic operations that do not require a transaction or row lock:

```sql
UPDATE playlist.playlist
SET tags = array_append(tags, :new_tag)
WHERE id = :playlist_id AND NOT (:new_tag = ANY(tags));

UPDATE playlist.playlist
SET tags = array_remove(tags, :tag_to_remove)
WHERE id = :playlist_id;
```

These are simpler, safer, and avoid the overhead of a `FOR UPDATE` lock. The candidate should
explain why the locked read-modify-write approach is preferred over atomic array operations.

### 6. Client-side sort across pages is unaddressed for track sorting

For playlist search (Sub-project 1), the candidate acknowledges that sorting applies only to
the current page. But for track sorting in Playlist.tsx (Sub-project 3), the same problem
exists — playlist tracks are paginated, so sorting by "Artist A-Z" only sorts the current
page, not the full playlist. The candidate should address this limitation explicitly in the
track sorting section.

---

## Clarification Questions

### Sub-Project 1 — Playlist Search

1. The existing `/1/playlist/search` endpoint calls `search_playlist()`, not
   `search_playlists_for_user()`. Your proposal says you'll extend the existing endpoint with
   an optional `user_name` parameter. What happens to the default behavior when `user_name` is
   absent — does it still search globally, or does it become a no-op without a user?

2. You clear the search and "instantly restore the original playlist layout by pulling from the
   initial loader data." What if the user was on page 5 of their playlists when they typed a
   search and then cleared it? Do they return to page 5, or to page 1?

### Sub-Project 2 — Playlist Tags

3. The Tag Manager Modal supports "rename." Walk me through how renaming a tag from "workout"
   to "gym" works at the API level. What endpoint does it call, and what does that endpoint do
   if the user has 40 playlists tagged "workout"?

4. Your proposal says tags are "personal organization" but they are stored on the shared
   playlist row. If user A (collaborator) adds tag "secret" to a collaborative playlist, does
   user B (creator) see "secret" in their tag filter bar? Is this the intended behavior?

5. You propose a `SELECT ... FOR UPDATE` transaction for the read-modify-write tag update.
   PostgreSQL has `array_append(tags, :tag)` and `array_remove(tags, :tag)` as single atomic
   statements. What does the locked approach give you that atomic array operations don't?

### Sub-Project 3 — Track Sorting

6. Your "Release Date" sort option — where does the release date come from? Look at the JSPF
   track fields in `listenbrainz/db/model/playlist.py` and tell me which field stores the
   release date, or whether it exists at all.

7. If a playlist has 500 tracks across 10 pages and the user selects "Sort by Artist A-Z",
   does the sort apply to all 500 tracks or only the ~50 on the current page?

### Sub-Project 4 — MB Collections

8. The `editor_collection` table has an `editor INTEGER` column. The LB user has
   `musicbrainz_id` (a string username) and `musicbrainz_row_id` (an integer). Which one maps
   to `editor_collection.editor`, and where in your code does this lookup happen?

9. Your `mb_collection_prefs` table stores `(user_id, collection_mbid, is_hidden)`. If a user
   hides a collection and then deletes their MB account, what happens to the rows in this
   table? How does the cascade work given the table references an external MB UUID, not a local
   FK?

---

## Comparison with the Competing Proposal (Gopal)

Both proposals address the same four features. Key differences worth noting:

| Area | Shreshth | Gopal |
|------|----------|-------|
| Execution order | 1→3→2→4 (strategic, smaller PRs first) | 1→2→3→4 |
| Tag design | Playlist-level shared tags | Per-user private tags (contradicts his own constraint) |
| OR tag logic | Explicitly covered (`&&` operator) | Not mentioned |
| Backend sort | Client-side only (simpler, correct) | Proposes backend ORDER BY (hits unresolvable `additional_metadata` issue) |
| Tag rename | UI exists but endpoints missing | Not mentioned |
| MB OAuth approach | Not described | Not described |
| MB editor ID | Not mentioned | Not mentioned |
| Collection hiding | `mb_collection_prefs` table | Not mentioned |
| Performance awareness | Explicit DOM virtualization note | Not mentioned |
| Code depth | High-level design, less pseudocode | More pseudocode (has bugs) |

Shreshth's proposal is architecturally sounder. The main gaps are the missing rename endpoint,
the Release Date sort with no data source, and the MB editor ID lookup.
