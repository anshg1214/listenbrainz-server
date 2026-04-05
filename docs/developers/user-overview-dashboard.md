# User Overview Dashboard — Proposal

## Background

ListenBrainz provides a rich set of features to users, but many go undiscovered because the
current user index page (`/user/:username/`) is a plain chronological listen log. Users who
land there have no visibility into auto-generated playlists, statistics, social features, Year
in Music, or the Explore tools.

This document proposes a dedicated **Overview Dashboard** page that consolidates all offerings
into a single scannable surface.

---

## Current Feature Inventory

### User Pages

| Route | What it Shows |
|---|---|
| `/user/:username/` (index) | Recent listens, playing now, listen count, social panel, pinned recording |
| `/user/:username/stats/` | Listening activity, top entities, daily heatmap, artist map, genre/era/evolution activity, album collage |
| `/user/:username/stats/top-artists/` | Full artist chart with pagination |
| `/user/:username/stats/top-albums/` | Full album chart |
| `/user/:username/stats/top-tracks/` | Full track chart |
| `/user/:username/taste/` | Loved/hated tracks, pinned recordings history |
| `/user/:username/playlists/` | User-created and collaborative playlists |
| `/user/:username/recommendations/` | AI-generated playlists (Daily Jams, Weekly Jams, Weekly Exploration, Top Discoveries, Missed Tracks) |
| `/user/:username/year-in-music/:year/` | Annual listening wrap-up |

### Statistics Computed (per-user, with range filtering)

- Top artists / releases / release groups / recordings
- Listening activity (bar chart / calendar heatmap)
- Daily activity heatmap (by hour and weekday)
- Artist map (choropleth by country)
- Artist activity
- Era activity (which decade the music is from)
- Genre activity
- Artist evolution (timeline of newly discovered artists)

**Available ranges**: `this_week`, `this_month`, `this_year`, `week`, `month`, `quarter`,
`year`, `half_yearly`, `all_time`

### Auto-Generated Playlists (Troi + Spark)

| Slug | Schedule | Description |
|---|---|---|
| `daily-jams` | Every midnight (per user timezone) | Personalized daily playlist |
| `weekly-jams` | Every Monday | Top recordings not recently heard |
| `weekly-exploration` | Every Monday | Newly discovered recordings |
| `top-discoveries-for-year` | Yearly | Best discoveries of the year |
| `top-missed-recordings-for-year` | Yearly | Popular tracks the user missed |

### Social Features

- Followers / following
- Similar users (computed by listening overlap)
- Compatibility score with other users
- User activity feed (`/recent/`)

### Explore Tools (`/explore/`)

- LB Radio (generative radio)
- Music Neighborhood (artist similarity graph)
- HueSound (music by color)
- Fresh Releases
- Art Creator (album grid visualizations)
- Cover Art Collage
- Similar Users explorer
- AI Brainz

### Other Features Rarely Discovered

- Embeddable "playing now" and "pin" widgets
- Spotify export for auto-generated playlists
- Atom syndication feeds
- Love/hate feedback
- Do Not Recommend
- Collaborative playlists

---

## The Problem

The current index page (`/user/:username/`) is a **chronological listen log** — nothing more.
Users who land there see a paginated stream of songs with no indication that ListenBrainz:

- Generates 5 different AI playlists for them every week
- Calculates which countries their music comes from
- Shows how their taste evolved over years
- Has a similarity graph of listeners like them
- Provides a full Year in Music report

Even the nav tabs (`Listens | Stats | Taste | Playlists | Created for you`) are low-information
labels — a new user cannot tell what "Taste" means without clicking.

---

## Proposed Solution: User Overview Dashboard

### New Route

Add `/user/:username/overview/` as a new tab (least disruption). After gathering feedback,
promote it to the default index and move the listen log to `/user/:username/listens/`.

---

### UI Layout

```
+------------------------------------------------------------------+
|  [Username]  Overview | Listens | Stats | Taste | Playlists |    |
|                       Created for you                             |
+------------------------------------------------------------------+
|  Hi, {username}                                                   |
+------------------------------------------------------------------+

+---------------------------+--------------------------------------+
|  Listening Snapshot       |  This [Week v] <- time selector      |
|                           |                                      |
|  [Pinned track card]      |  412 listens                         |
|                           |  67 artists                          |
|  42,891 total listens     |  [bar chart]                         |
|  Member since 2020        |  [-> Full Stats]                     |
+---------------------------+--------------------------------------+

+----------------------+-------------------------------------------+
|  Social              |  Top Charts (This Month) <- selector      |
|                      |                                           |
|  23 Followers        |  Artists:       Tracks:                   |
|  41 Following        |  1. Hozier      1. Too Sweet              |
|  12 Similar users    |  2. Mitski      2. Nobody                 |
|  [View similar]      |  3. Phoebe B.   3. Moon Song              |
|  [Activity feed]     |                                           |
|                      |  [-> Full stats]                          |
+----------------------+-------------------------------------------+

+------------------------------------------------------------------+
|  Your Taste                                                      |
|  127 loved tracks    12 hated tracks    1 pinned                 |
|  [-> See full taste]                                             |
+------------------------------------------------------------------+

+----------------------------+     +------------------------------+
|  Created for You           |     |  Explore                     |
|  (horizontal scroll ->)    |     |                              |
|  +----------+ +----------+ |     |  > LB Radio                  |
|  | Daily    | | Weekly   | |     |  > Fresh Releases            |
|  | Jams     | | Jams     | |     |  > Art Creator               |
|  | 50 tracks| | 50 tracks| |     |  > Music Neighborhood        |
|  | timer    | | timer    | |     |  > HueSound                  |
|  +----------+ +----------+ |     |  [-> Explore all tools]      |
+----------------------------+     +------------------------------+
```

---

### Widget Specification

#### Row 1 — Listening Snapshot (left, ~40%) + Activity Widget (right, ~60%)

**Listening Snapshot**
- Background / featured image: most recent year's YIM album cover art
  (`generateAlbumArtThumbnailLink(caa_id, caa_release_mbid)` → 500px thumb)
- Small row of year covers for past years (reuse pattern from `YIMYearSelection.tsx`),
  each linking to that year's `/year-in-music/:year/` page
- Total listen count overlaid or below
- Member since date
- Pinned recording card (if exists)
- Currently playing (live websocket update)
- **Data**: `get_yim_covers_for_user` (returns list of `{ year, caa_id, caa_release_mbid }`)
  + current `profile()` view data + websockets

**This [Period] Activity** — with time period selector (Week / Month / Year / All time)
- Total listens for the period
- Unique artists for the period
- Bar chart (reuse `UserListeningActivity` component, compact variant)
- Link to `/stats/?range=<selected>`
- **Data**: `/1/stats/user/:user/listening-activity?range=<range>`

#### Row 2 — Social Panel (left, ~30%) + Top Charts (right, ~70%)

**Social Panel**
- Follower count, following count, similar user count
- Links: "View similar users", "Activity feed"
- **Data**: `/1/user/:user/followers` + `/1/user/:user/following` +
  `get_similar_users` (count only, no need to load full list)

**Top Charts** — with time period selector (This Week / This Month / This Year / All time)
- Two columns side-by-side: Top 5 Artists | Top 5 Tracks
- Each entry is a simple ranked row: `#N  Name  · listen_count`
- No ListenCard — no cover art, no controls menu, no timestamps
- Artist name links to `/artist/:mbid/`, track name links to `/track/:mbid/`
- Links to `/stats/top-artists/?range=<range>` and `/stats/top-tracks/?range=<range>`
- **Data**: `/1/stats/user/:user/artists?range=<range>&count=5` +
  `/1/stats/user/:user/recordings?range=<range>&count=5`

#### Row 3 — Taste Summary (full width)
- Count of loved tracks, hated tracks, pinned recordings
- Link to `/taste/`
- **Data**: `get_feedback_count_for_user` + `get_pin_count_for_user`

#### Row 4 — Created for You (left, ~65%) + Explore Tools (right, ~35%)

**Created for You** — horizontal scroll
- Cards for each auto-generated playlist (reuse card style from `RecommendationsPage.tsx`)
- Shows: type label (Daily Jams / Weekly Jams / Weekly Exploration / etc.), track count,
  expiry countdown timer
- Play button on each card
- Link to `/recommendations/`
- **Data**: existing recommendations endpoint (no new API needed)

**Explore Tools**
- List links: LB Radio, Fresh Releases, Art Creator, Music Neighborhood, HueSound
- Each with a one-line description
- "Explore all tools" link to `/explore/`

---

## Backend Changes

### New Aggregated Endpoint

Add to `listenbrainz/webserver/views/user.py`:

```python
@user_bp.post("/<mb_username:user_name>/overview/")
def overview(user_name: str):
    """Returns aggregated overview data for the user dashboard."""
```

Response shape:

```json
{
  "user": { "id": 1, "name": "username", "created": "2020-01-01" },
  "listen_count": 42891,
  "playing_now": { "...": "..." },
  "current_pin": { "...": "..." },
  "followers_count": 23,
  "following_count": 41,
  "similar_users": [ "top 5 similar users" ],
  "feedback_counts": { "love": 127, "hate": 12 },
  "pin_count": 1,
  "recommendation_playlists": [ "summary objects, no tracks" ],
  "yim_covers": [ { "year": 2024, "caa_id": "...", "caa_release_mbid": "..." } ]
}
```

Stats (this week activity, top charts, artist map) load lazily from existing endpoints on the
client side, since they are heavier Spark-backed data. This keeps the overview endpoint fast.

---

## Frontend Changes

### New Files

| File | Purpose |
|---|---|
| `frontend/js/src/user/overview/Overview.tsx` | Main overview page component |
| `frontend/js/src/user/overview/widgets/ListeningSnapshot.tsx` | YIM album covers, listen count, member since, pinned recording, playing now |
| `frontend/js/src/user/overview/widgets/ActivityWidget.tsx` | Listening activity bar chart with time period selector |
| `frontend/js/src/user/overview/widgets/SocialPanel.tsx` | Follower/following counts, links to feed and similar users |
| `frontend/js/src/user/overview/widgets/TopChartsPreview.tsx` | Top 5 artists + tracks with time period selector |
| `frontend/js/src/user/overview/widgets/TasteSummary.tsx` | Loved/hated counts, pin count |
| `frontend/js/src/user/overview/widgets/RecommendationsPreview.tsx` | Horizontal scroll of auto-generated playlist cards |
| `frontend/js/src/user/overview/widgets/ExploreTools.tsx` | Quick links to Explore features |

### Routing Changes

**`frontend/js/src/user/routes/userRoutes.tsx`** — add overview route and explicit listens path:

```tsx
{
  index: true,
  element: <Navigate to="overview/" replace />,
},
{
  path: "overview/",
  lazy: {
    loader: async () => RouteQueryLoader("overview", true),
    Component: async () => (await import("../overview/Overview")).default,
  },
},
{
  path: "listens/",
  lazy: {
    loader: async () => RouteQueryLoader("dashboard", true),
    Component: async () => (await import("../Dashboard")).default,
  },
},
```

**`frontend/js/src/user/layout.tsx`** — add Overview and Listens tabs:

```tsx
<NavItem label="Overview" url={`/user/${escapedUserName}/overview/`} ... />
<NavItem label="Listens"  url={`/user/${escapedUserName}/listens/`}  ... />
<NavItem label="Stats"    ... />
// ... rest unchanged
```

---

## Implementation Phases

### Phase 1 — New Tab (no disruption)
- Add `/user/:username/overview/` route and nav tab
- Create the Overview page with lazy-loaded widgets
- Create the aggregated `/user/:username/overview/` backend endpoint
- Stats widgets load independently with skeleton placeholders

### Phase 2 — Polish and Promote
- Gather user feedback
- Add "first visit" tooltips ("Did you know we generate playlists for you?")
- Make Overview the default index page
- Move listen log to `/user/:username/listens/`
- Update all internal links

### Phase 3 — Personalization
- Let users customize which widgets appear
- Add notification badges to nav items when new playlists are ready
- Add trend indicators ("23% more listens than last week")

---

## Design Decisions

1. **Listens page stays** — do not remove it, just demote from index position. Power users
   rely on the listen log directly.

2. **Stats load lazily** — the overview endpoint returns social/playlists/listen-count
   immediately. Spark-backed stats trigger separate async calls with skeleton placeholders,
   matching the pattern already used in the stats page.

3. **Bento grid, not tabs** — everything visible at a glance. Each widget has a clear
   "full page" link. No information is hidden behind a click-to-reveal.

4. **Works for other users too** — when viewing another user's overview, Explore/Taste
   sections show that user's data. Recommendation playlists section is hidden (private data).

5. **Mobile-first** — bento grid collapses to single column on small screens. Horizontal
   scroll sections get touch-friendly swipe behavior.

---

## What Requires No New Backend Work

All of the following already exist and need no schema changes:

- Stats APIs: `/1/stats/user/:user/artists`, `/recordings`, `/listening-activity`
- Social APIs: `/1/user/:user/followers`, `/1/user/:user/following`
- Playlists: recommendations endpoint already returns summary data
- Feedback counts: `get_feedback_count_for_user` in `listenbrainz/db/feedback.py`
- Pin counts: `get_pin_count_for_user` in `listenbrainz/db/pinned_recording.py`
- Similar users count: `get_similar_users` in `listenbrainz/db/user.py`
- `UserListeningActivity` component: reusable with compact variant for the activity widget
- Playlist card styles: `frontend/js/src/user/recommendations/RecommendationsPage.tsx`
