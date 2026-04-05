# XSPF Playlist Import Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Allow users to upload an XSPF playlist file and import it into ListenBrainz, resolving track names to MusicBrainz recording MBIDs via the troi pipeline.

**Architecture:** Parse XSPF on the backend, resolve unidentified tracks using troi's `mbid_mapping_tracks()`, enrich with `RecordingLookupElement`, then return JSPF to the frontend which creates the playlist via the existing endpoint.

**Tech Stack:** Python (`xml.etree.ElementTree`, troi), Flask, React/TypeScript, NiceModal

---

## Repositories

- **troi:** `/Users/ansh/Desktop/Ansh/gsoc/troi-recommendation-playground/`
- **listenbrainz-server:** `/Users/ansh/Desktop/Ansh/gsoc/listenbrainz-server/`

---

## Background: How JSPF Import Works Today

Understanding the existing JSPF import end-to-end is essential before touching anything. XSPF import will follow the same flow.

### What is JSPF?

JSPF is the JSON equivalent of XSPF. ListenBrainz uses it internally as the playlist wire format. A JSPF playlist looks like:

```json
{
  "playlist": {
    "title": "My Playlist",
    "annotation": "A description",
    "extension": {
      "https://musicbrainz.org/doc/jspf#playlist": {
        "public": true
      }
    },
    "track": [
      {
        "identifier": ["https://musicbrainz.org/recording/e8f9b188-f819-4e43-ab0f-4bd26ce9ff56"],
        "title": "Never Gonna Give You Up",
        "creator": "Rick Astley"
      }
    ]
  }
}
```

Every track **must** have a `identifier` that is a full MusicBrainz recording URI. The backend rejects any track without one.

### JSPF import flow

**Frontend** (`frontend/js/src/user/playlists/components/ImportJSPFPlaylistModal.tsx`):
1. User picks a `.jspf` or `.json` file.
2. Browser reads it with `FileReader` and parses it as JSON.
3. The parsed `JSPFObject` is passed directly to `APIService.createPlaylist()`.
4. `createPlaylist()` POSTs to `POST /api/v1/playlist/create` with the JSPF as the body.

**Backend** (`listenbrainz/webserver/views/playlist_api.py`, function `create_playlist`):
1. Validates the JSPF structure (`validate_create_playlist_required_items`, `validate_playlist`).
2. `validate_playlist` calls `get_track_recording_mbid()` on every track and raises 400 if any track lacks a valid MB recording URI.
3. Inserts the playlist and recordings into the database via `db_playlist.create()`.
4. Returns `{"status": "ok", "playlist_mbid": "<uuid>"}`.

**Key constraint:** JSPF import only works because the JSPF file already contains MBIDs for every track. A user who exports a JSPF from ListenBrainz and re-imports it gets a perfect round-trip. A file from an external tool that lacks MBIDs would be rejected.

### How Music Service Imports Solve This

Spotify, Apple Music, and SoundCloud don't provide MBIDs. ListenBrainz handles this with a two-step flow:

**Step 1 — Resolve tracks to MBIDs (new backend endpoint):**

The frontend calls a service-specific endpoint (e.g. `GET /api/v1/playlist/spotify/<id>/tracks`) which:
1. Calls `import_from_spotify()` in `listenbrainz/troi/import_ms.py`.
2. That function instantiates `ImportPlaylistPatch` from troi with the user's Spotify token.
3. The troi pipeline: `RecordingsFromMusicServiceElement` → `RecordingLookupElement` → `PlaylistMakerElement`
   - `RecordingsFromMusicServiceElement` calls `music_service_tracks_to_mbid()` in `troi/tools/common_lookup.py`.
   - `music_service_tracks_to_mbid()` fetches track names and artist names from Spotify, then POSTs them in batches of 50 to `https://api.listenbrainz.org/1/metadata/lookup/` which returns the best-matching recording MBID for each.
   - `RecordingLookupElement` enriches those MBIDs with full metadata (artist credit, release name, etc.) from `https://api.listenbrainz.org/1/metadata/recording`.
   - `PlaylistMakerElement` assembles a troi `Playlist` object.
4. The patch calls `.get_jspf()` on the result — this returns a complete JSPF dict where every track now has a MB recording URI.

**Step 2 — Create the playlist:**

The frontend receives the JSPF and calls `APIService.createPlaylist()` exactly as in the JSPF import. From here on, the flow is identical.

### Troi resolution utilities involved

| File | Function | Purpose |
|------|----------|---------|
| `troi/tools/common_lookup.py` | `mbid_mapping_tracks(track_lists)` | POSTs `[{recording_name, artist_name}]` to the LB metadata lookup API, returns list of MBIDs |
| `troi/tools/common_lookup.py` | `music_service_tracks_to_mbid(...)` | Fetches tracks from a music service then calls `mbid_mapping_tracks` |
| `troi/musicbrainz/recording_lookup.py` | `RecordingLookupElement` | Enriches `Recording(mbid=...)` objects with full metadata |
| `troi/playlist.py` | `RecordingsFromMusicServiceElement` | Wraps `music_service_tracks_to_mbid()` as a troi pipeline Element |
| `troi/playlist.py` | `PlaylistMakerElement` | Assembles the final `Playlist` object from `Recording` list |
| `troi/patches/playlist_from_ms.py` | `ImportPlaylistPatch` | Wires the pipeline together for music service imports |

---

## How XSPF Import Will Work

### What is XSPF?

XSPF ("XML Shareable Playlist Format") is the XML predecessor to JSPF. It is the format exported by tools like Soundiiz, which lets users export playlists from any streaming service.

We need to support two real-world XSPF sources:

**Source 1 — Soundiiz export** (actual file examined):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>Exported XSPF Playlist</title>
  <creator>Soundiiz.com</creator>
  <trackList>
    <track>
      <title>Pink Venom</title>
      <creator>BLACKPINK</creator>
      <album>Pink Venom</album>
      <isrc>KRA402200017</isrc>
      <platform>spotify</platform>
      <duration>187</duration>
      <url>https://open.spotify.com/track/0skYUMpS0AcbpjcGsAbRGj</url>
    </track>
  </trackList>
</playlist>
```

**Source 2 — ListenBrainz XSPF export** (round-trip, from `GET /1/playlist/<mbid>/xspf`):

```xml
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>Copy of Top discoveries of 2020</title>
  <identifier>https://listenbrainz.org/playlist/862add71-...</identifier>
  <annotation>&lt;p&gt;Some description&lt;/p&gt;</annotation>
  <extension application="https://musicbrainz.org/doc/jspf#playlist">
    <public>true</public>
    <creator>chaban</creator>
  </extension>
  <trackList>
    <track>
      <identifier>https://musicbrainz.org/recording/85701f84-c37f-4857-89d3-29625f02d943</identifier>
      <creator>Frozen Starfall feat. Milkychan</creator>
      <album>Mirrored Worlds</album>
      <title>Inner Storm</title>
      <extension application="https://musicbrainz.org/doc/jspf#track">
        <added_by>troi-bot</added_by>
        <artist_identifiers>
          <identifier>https://musicbrainz.org/artist/e4b67829-...</identifier>
        </artist_identifiers>
      </extension>
    </track>
  </trackList>
</playlist>
```

Key observations across both formats:

- **Soundiiz: no `<identifier>` per track** — all resolution goes through `artist_name` + `recording_name`.
- **LB export: `<identifier>` with MB recording URI** — MBID can be used directly; no resolution needed.
- **LB export: nested artist `<identifier>` elements** — inside `<extension>/<artist_identifiers>/`, not direct children of `<track>`. `ElementTree.findtext(tag)` with a bare tag name only searches direct children, so these are never confused with the recording identifier. (Verified by test.)
- **LB export: playlist-level `<identifier>`** — the LB playlist URI on the `<playlist>` element, not a track field. Ignored by the parser.
- **`<isrc>` present in Soundiiz** — every Soundiiz track carries an ISRC. The LB metadata lookup API does not currently support ISRC lookup, so we store it for future use but do not use it for resolution.
- **Malformed HTML entities in Soundiiz titles** — Soundiiz double-escapes special characters: `&lt;#039;` → `'`, `&lt;amp;` → `&`. `xml.etree.ElementTree` parses `&lt;` as `<`, leaving literal garbage like `I Ain<#039;t Worried`. The parser must clean these before passing titles to the lookup API. LB exports use correct XML escaping and need no cleanup.
- **All elements in XSPF namespace** — because the default namespace `xmlns="http://xspf.org/ns/0/"` applies to all descendants. Access with `findtext(f"{{{XSPF_NS}}}tag")` — this applies to both standard fields (`<title>`, `<creator>`) and Soundiiz extensions (`<isrc>`, `<platform>`).

XSPF uses the namespace `http://xspf.org/ns/0/` on all elements.

### The Problem

XSPF tracks from Soundiiz have no MBIDs — just `<title>` and `<creator>`. The existing JSPF import endpoint would reject this. We need the same name-to-MBID resolution that music service imports use.

### The Solution: Mirror the Music Service Import Pattern

XSPF import uses **exactly** the same two-step flow as Spotify/Apple/SoundCloud, just with a different data source:

```
User uploads .xspf file
        │
        ▼
Frontend reads file as text (FileReader)
        │
        ▼
POST /api/v1/playlist/xspf/tracks   ← new endpoint, receives raw XML
        │
        ├─ Parse XSPF with xml.etree.ElementTree
        │
        ├─ Tracks WITH <identifier> containing MB recording URI
        │       └─ Use MBID directly → Recording(mbid=...)
        │
        └─ Tracks WITHOUT MBID
                └─ batch POST to LB metadata/lookup API (mbid_mapping_tracks)
                        └─ Recording(mbid=resolved_mbid)
        │
        ▼
RecordingLookupElement enriches all recordings
        │
        ▼
PlaylistMakerElement assembles Playlist
        │
        ▼
Returns JSPF JSON to frontend
        │
        ▼
Frontend calls createPlaylist() ← same as JSPF import from here
        │
        ▼
Playlist created in ListenBrainz
```

### New Code

#### In troi (`troi-recommendation-playground`)

**`troi/tools/xspf_lookup.py`** — pure XML parsing, no network calls:
```
parse_xspf(xspf_content) -> (name: str, description: str, tracks: list[dict])
```
Each dict has `recording_name`, `artist_name`, and optionally `recording_mbid`.

**`troi/playlist.py`** — new Element class `RecordingsFromXSPFElement`:
- Takes raw XSPF XML string
- Calls `parse_xspf()` to extract tracks
- Tracks with MBIDs → `Recording(mbid=...)` directly
- Tracks without MBIDs → batched through `mbid_mapping_tracks()` → `Recording(mbid=...)`
- Returns `list[Recording]`

**`troi/patches/playlist_from_xspf.py`** — new Patch `ImportXSPFPlaylistPatch`:
```
RecordingsFromXSPFElement → RecordingLookupElement → PlaylistMakerElement
```

#### In listenbrainz-server

**`listenbrainz/troi/import_ms.py`** — `import_from_xspf(xspf_content, user_token)`:
- Instantiates `ImportXSPFPlaylistPatch`, runs it, returns JSPF dict. Mirrors `import_from_spotify()`.

**`listenbrainz/webserver/views/playlist_api.py`** — `POST /api/v1/playlist/xspf/tracks`:
- Reads raw XML from request body
- Calls `import_from_xspf()`
- Returns JSPF as JSON

**`frontend/js/src/utils/APIService.ts`** — `importXSPFPlaylistTracks(userToken, xspfContent)`:
- POSTs XSPF XML to the new endpoint, returns JSON

**`frontend/js/src/user/playlists/components/ImportXSPFPlaylistModal.tsx`** — new React modal:
- File input accepting `.xspf`
- Reads file content, calls `importXSPFPlaylistTracks`, then `createPlaylist`, then resolves the NiceModal promise

**`frontend/js/src/user/playlists/Playlists.tsx`** — add "Upload XSPF file" item to the Import dropdown

---

## Tasks

---

### Task 1: XSPF XML parser utility in troi

**Files:**
- Create: `troi/tools/xspf_lookup.py`
- Create: `tests/test_xspf_lookup.py`

**Background on quirks to handle:**

Real Soundiiz XSPF files have three issues the parser must handle:

1. **No `<identifier>`** — tracks have no identifier at all; resolution always falls through to name matching.
2. **Malformed HTML entities** — Soundiiz double-escapes special characters: `&lt;#039;` → `'`, `&lt;amp;` → `&`, `&lt;quot;` → `"`. ElementTree parses these literally, producing garbage titles. We must clean them post-parse.
3. **`<isrc>` extension field** — present on every Soundiiz track and in the XSPF namespace (inherits the default namespace from `<playlist>`). Store it for potential future use; do not use for resolution (the LB lookup API does not support ISRC).

**Step 1: Write the failing test**

Three holistic tests — each exercises the full parse output for a complete fixture, not individual fields in isolation.

```python
# Create: tests/test_xspf_lookup.py

from troi.tools.xspf_lookup import parse_xspf

# Real Soundiiz export shape: no <identifier>, has <isrc>, malformed entities
XSPF_SOUNDIIZ = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>My Soundiiz Playlist</title>
  <trackList>
    <track>
      <title>Pink Venom</title>
      <creator>BLACKPINK</creator>
      <album>Pink Venom</album>
      <isrc>KRA402200017</isrc>
    </track>
    <track>
      <title>I Ain&lt;#039;t Worried</title>
      <creator>OneRepublic</creator>
      <isrc>USUM72206227</isrc>
    </track>
    <track>
      <title>Bad Decisions (with BTS &lt;amp; Snoop Dogg)</title>
      <creator>benny blanco, BTS, Snoop Dogg</creator>
      <isrc>USUM72210832</isrc>
    </track>
  </trackList>
</playlist>"""

# LB XSPF export shape: recording <identifier> + nested artist identifiers + annotation
XSPF_LB_EXPORT = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>Top discoveries</title>
  <identifier>https://listenbrainz.org/playlist/862add71-2e87-44b7-abe1-a7f1a8658346</identifier>
  <annotation>&lt;p&gt;Some description&lt;/p&gt;</annotation>
  <extension application="https://musicbrainz.org/doc/jspf#playlist">
    <public>true</public>
    <creator>chaban</creator>
  </extension>
  <trackList>
    <track>
      <identifier>https://musicbrainz.org/recording/85701f84-c37f-4857-89d3-29625f02d943</identifier>
      <creator>Frozen Starfall feat. Milkychan</creator>
      <album>Mirrored Worlds</album>
      <title>Inner Storm</title>
      <extension application="https://musicbrainz.org/doc/jspf#track">
        <added_by>troi-bot</added_by>
        <artist_identifiers>
          <identifier>https://musicbrainz.org/artist/e4b67829-4010-4420-8d13-125266683b77</identifier>
        </artist_identifiers>
      </extension>
    </track>
    <track>
      <identifier>https://example.com/not-a-mb-uri</identifier>
      <creator>Darude</creator>
      <title>Sandstorm</title>
    </track>
  </trackList>
</playlist>"""


def test_parse_soundiiz_xspf():
    """Full parse of a Soundiiz-style export: entity cleanup, ISRC extraction, no MBIDs."""
    name, description, tracks = parse_xspf(XSPF_SOUNDIIZ)

    assert name == "My Soundiiz Playlist"
    assert description == ""
    assert len(tracks) == 3

    assert tracks[0] == {"recording_name": "Pink Venom", "artist_name": "BLACKPINK", "isrc": "KRA402200017"}
    assert tracks[1] == {"recording_name": "I Ain't Worried", "artist_name": "OneRepublic", "isrc": "USUM72206227"}
    assert tracks[2] == {"recording_name": "Bad Decisions (with BTS & Snoop Dogg)",
                         "artist_name": "benny blanco, BTS, Snoop Dogg", "isrc": "USUM72210832"}


def test_parse_lb_xspf_roundtrip():
    """Full parse of an LB XSPF export: recording MBIDs, annotation, non-MB identifiers ignored,
    nested artist identifiers not confused with recording identifier."""
    name, description, tracks = parse_xspf(XSPF_LB_EXPORT)

    assert name == "Top discoveries"
    assert description == "<p>Some description</p>"  # LB uses correct XML escaping
    assert len(tracks) == 2

    # Track with MB recording URI: MBID extracted directly
    assert tracks[0]["recording_name"] == "Inner Storm"
    assert tracks[0]["artist_name"] == "Frozen Starfall feat. Milkychan"
    assert tracks[0]["recording_mbid"] == "85701f84-c37f-4857-89d3-29625f02d943"
    assert "isrc" not in tracks[0]

    # Track with non-MB identifier: recording_mbid must be absent, falls through to name resolution
    assert tracks[1]["recording_name"] == "Sandstorm"
    assert "recording_mbid" not in tracks[1]


def test_parse_empty_xspf():
    """Empty trackList and missing optional fields produce safe defaults."""
    xspf = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/"><trackList/></playlist>"""

    name, description, tracks = parse_xspf(xspf)
    assert name == "Untitled Playlist"
    assert description == ""
    assert tracks == []
```

**Step 2: Run to confirm it fails**

```bash
cd /Users/ansh/Desktop/Ansh/gsoc/troi-recommendation-playground
python -m pytest tests/test_xspf_lookup.py -v
```
Expected: `ModuleNotFoundError: No module named 'troi.tools.xspf_lookup'`

**Step 3: Write the implementation**

Create `troi/tools/xspf_lookup.py`:

```python
from xml.etree import ElementTree as ET

XSPF_NS = "http://xspf.org/ns/0/"
MB_RECORDING_URI_PREFIX = "https://musicbrainz.org/recording/"

# Soundiiz double-escapes HTML entities. After ElementTree parsing these appear
# as literal character sequences in text. Map them back to the intended character.
_ENTITY_FIXES = [
    ("<#039;", "'"),
    ("<amp;",  "&"),
    ("<quot;", '"'),
    ("<lt;",   "<"),
    ("<gt;",   ">"),
]


def _fix_entities(text: str) -> str:
    """Clean up Soundiiz-style double-escaped HTML entities."""
    for broken, replacement in _ENTITY_FIXES:
        text = text.replace(broken, replacement)
    return text


def parse_xspf(xspf_content: str):
    """Parse XSPF XML content.

    Returns:
        tuple: (name, description, tracks)
            name (str): playlist title, defaults to "Untitled Playlist"
            description (str): playlist annotation, defaults to ""
            tracks (list[dict]): each dict has:
                - recording_name (str)
                - artist_name (str)
                - isrc (str, optional) — present when <isrc> is in the file;
                  stored for future use, not currently used for MBID resolution
                - recording_mbid (str, optional) — only set when <identifier>
                  contains a MusicBrainz recording URI; absent in most
                  real-world XSPF files (e.g. Soundiiz exports)
    """
    root = ET.fromstring(xspf_content)

    name = root.findtext(f"{{{XSPF_NS}}}title") or "Untitled Playlist"
    description = root.findtext(f"{{{XSPF_NS}}}annotation") or ""

    tracks = []
    track_list = root.find(f"{{{XSPF_NS}}}trackList")
    if track_list is None:
        return name, description, tracks

    for track in track_list.findall(f"{{{XSPF_NS}}}track"):
        recording_name = _fix_entities(track.findtext(f"{{{XSPF_NS}}}title") or "")
        artist_name = _fix_entities(track.findtext(f"{{{XSPF_NS}}}creator") or "")

        track_data = {
            "recording_name": recording_name,
            "artist_name": artist_name,
        }

        isrc = track.findtext(f"{{{XSPF_NS}}}isrc")
        if isrc:
            track_data["isrc"] = isrc

        identifier = track.findtext(f"{{{XSPF_NS}}}identifier")
        if identifier and identifier.startswith(MB_RECORDING_URI_PREFIX):
            track_data["recording_mbid"] = identifier[len(MB_RECORDING_URI_PREFIX):]

        tracks.append(track_data)

    return name, description, tracks
```

**Step 4: Run tests to confirm they pass**

```bash
python -m pytest tests/test_xspf_lookup.py -v
```
Expected: 3 tests PASS.

**Step 5: Commit**

```bash
cd /Users/ansh/Desktop/Ansh/gsoc/troi-recommendation-playground
git add troi/tools/xspf_lookup.py tests/test_xspf_lookup.py
git commit -m "feat: add XSPF parser with entity cleanup and ISRC extraction"
```

---

### Task 2: RecordingsFromXSPFElement in troi

**Files:**
- Modify: `troi/playlist.py` (add class after `RecordingsFromMusicServiceElement`, around line 612)

**Step 1: Write the failing test**

```python
# Add to a new file: tests/test_xspf_element.py

import pytest
from unittest.mock import patch
from troi.playlist import RecordingsFromXSPFElement

XSPF_MIXED = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>Mixed</title>
  <trackList>
    <track>
      <title>Blue (Da Ba Dee)</title>
      <creator>Eiffel 65</creator>
    </track>
    <track>
      <title>Sandstorm</title>
      <creator>Darude</creator>
      <identifier>https://musicbrainz.org/recording/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa</identifier>
    </track>
  </trackList>
</playlist>"""


def test_recordings_from_xspf_element_mixed():
    """Tracks with MBIDs are used directly; tracks without are resolved."""

    def fake_mbid_mapping(track_lists):
        # Called with tracks_needing_resolution; return one MBID
        return ["bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb"]

    with patch("troi.tools.common_lookup.mbid_mapping_tracks", side_effect=fake_mbid_mapping):
        element = RecordingsFromXSPFElement(XSPF_MIXED)
        recordings = element.read([])

    assert len(recordings) == 2
    mbids = {r.mbid for r in recordings}
    assert "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa" in mbids
    assert "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb" in mbids


def test_recordings_from_xspf_element_all_mbids():
    """When all tracks have MBIDs, mbid_mapping_tracks is never called."""

    xspf = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <trackList>
    <track>
      <title>Sandstorm</title>
      <creator>Darude</creator>
      <identifier>https://musicbrainz.org/recording/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa</identifier>
    </track>
  </trackList>
</playlist>"""

    with patch("troi.tools.common_lookup.mbid_mapping_tracks") as mock_mapping:
        element = RecordingsFromXSPFElement(xspf)
        recordings = element.read([])
        mock_mapping.assert_not_called()

    assert len(recordings) == 1
    assert recordings[0].mbid == "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"


def test_recordings_from_xspf_element_empty():
    xspf = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/"><trackList/></playlist>"""

    with patch("troi.tools.common_lookup.mbid_mapping_tracks") as mock_mapping:
        element = RecordingsFromXSPFElement(xspf)
        recordings = element.read([])
        mock_mapping.assert_not_called()

    assert recordings == []
```

**Step 2: Run to confirm it fails**

```bash
cd /Users/ansh/Desktop/Ansh/gsoc/troi-recommendation-playground
python -m pytest tests/test_xspf_element.py -v
```
Expected: `ImportError: cannot import name 'RecordingsFromXSPFElement'`

**Step 3: Add `RecordingsFromXSPFElement` to `troi/playlist.py`**

Locate the end of the `RecordingsFromMusicServiceElement` class (around line 612) and insert the new class immediately after:

```python
class RecordingsFromXSPFElement(Element):
    """Create Recording objects from raw XSPF XML content.

    Tracks that carry a MusicBrainz recording URI in their <identifier>
    element are used directly. The remaining tracks are resolved to MBIDs
    via the ListenBrainz metadata lookup API (mbid_mapping_tracks).
    """

    def __init__(self, xspf_content: str):
        """
        Args:
            xspf_content: Raw XSPF XML string.
        """
        super().__init__()
        self.xspf_content = xspf_content

    @staticmethod
    def outputs():
        return [Recording]

    def read(self, inputs):
        from more_itertools import chunked
        from troi.tools.xspf_lookup import parse_xspf
        from troi.tools.common_lookup import mbid_mapping_tracks, MAX_LOOKUPS_PER_POST

        _, _, tracks = parse_xspf(self.xspf_content)

        recordings = []
        tracks_needing_resolution = []

        for track in tracks:
            if "recording_mbid" in track:
                recordings.append(Recording(mbid=track["recording_mbid"]))
            else:
                tracks_needing_resolution.append({
                    "recording_name": track["recording_name"],
                    "artist_name": track["artist_name"],
                })

        if tracks_needing_resolution:
            track_lists = list(chunked(tracks_needing_resolution, MAX_LOOKUPS_PER_POST))
            resolved_mbids = mbid_mapping_tracks(track_lists)
            for mbid in resolved_mbids:
                recordings.append(Recording(mbid=mbid))

        return recordings
```

**Step 4: Run tests to confirm they pass**

```bash
python -m pytest tests/test_xspf_element.py -v
```
Expected: 3 tests PASS.

**Step 5: Commit**

```bash
git add troi/playlist.py tests/test_xspf_element.py
git commit -m "feat: add RecordingsFromXSPFElement to troi pipeline"
```

---

### Task 3: ImportXSPFPlaylistPatch in troi

**Files:**
- Create: `troi/patches/playlist_from_xspf.py`

**Step 1: Write the failing test**

```python
# Create: tests/test_playlist_from_xspf_patch.py

import pytest
from unittest.mock import patch, MagicMock
from troi.patches.playlist_from_xspf import ImportXSPFPlaylistPatch

SAMPLE_XSPF = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>Soundiiz Export</title>
  <annotation>My road trip playlist</annotation>
  <trackList>
    <track>
      <title>Blue (Da Ba Dee)</title>
      <creator>Eiffel 65</creator>
    </track>
  </trackList>
</playlist>"""


def test_patch_slug():
    assert ImportXSPFPlaylistPatch.slug() == "import-xspf-playlist"


def test_patch_creates_pipeline():
    args = {
        "xspf_content": SAMPLE_XSPF,
        "token": "test-token",
        "upload": False,
        "created_for": None,
        "echo": False,
        "min_recordings": 1,
    }
    patch_obj = ImportXSPFPlaylistPatch(args)
    # create() should return a PlaylistMakerElement without raising
    result = patch_obj.create(args)
    assert result is not None
```

**Step 2: Run to confirm it fails**

```bash
python -m pytest tests/test_playlist_from_xspf_patch.py -v
```
Expected: `ModuleNotFoundError: No module named 'troi.patches.playlist_from_xspf'`

**Step 3: Create `troi/patches/playlist_from_xspf.py`**

```python
from troi import Playlist
from troi.patch import Patch
from troi.playlist import RecordingsFromXSPFElement, PlaylistMakerElement
from troi.musicbrainz.recording_lookup import RecordingLookupElement
from troi.tools.xspf_lookup import parse_xspf


class ImportXSPFPlaylistPatch(Patch):

    @staticmethod
    def inputs():
        """
        A patch that imports an XSPF playlist file into ListenBrainz.

        \b
        XSPF_CONTENT is the raw XML string of the XSPF file.
        """
        return [
            {"type": "argument", "args": ["xspf_content"], "kwargs": {"required": True}},
        ]

    @staticmethod
    def outputs():
        return [Playlist]

    @staticmethod
    def slug():
        return "import-xspf-playlist"

    @staticmethod
    def description():
        return "Import an XSPF playlist file into ListenBrainz"

    def create(self, inputs):
        xspf_content = inputs["xspf_content"]

        name, desc, _ = parse_xspf(xspf_content)

        source = RecordingsFromXSPFElement(xspf_content=xspf_content)

        rec_lookup = RecordingLookupElement()
        rec_lookup.set_sources(source)

        pl_maker = PlaylistMakerElement(name, desc, patch_slug=self.slug())
        pl_maker.set_sources(rec_lookup)

        return pl_maker
```

**Step 4: Run tests to confirm they pass**

```bash
python -m pytest tests/test_playlist_from_xspf_patch.py -v
```
Expected: 2 tests PASS.

**Step 5: Run full troi test suite to check for regressions**

```bash
python -m pytest tests/ -v --tb=short 2>&1 | tail -30
```
Expected: no regressions.

**Step 6: Commit**

```bash
git add troi/patches/playlist_from_xspf.py tests/test_playlist_from_xspf_patch.py
git commit -m "feat: add ImportXSPFPlaylistPatch"
```

---

### Task 4: Backend import function in listenbrainz-server

**Files:**
- Modify: `listenbrainz/troi/import_ms.py`

**Step 1: Write the failing test**

```python
# In listenbrainz-server, create: listenbrainz/troi/tests/test_import_ms_xspf.py

import pytest
from unittest.mock import patch, MagicMock

SAMPLE_XSPF = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>Test Playlist</title>
  <trackList>
    <track><title>Blue</title><creator>Eiffel 65</creator></track>
  </trackList>
</playlist>"""

EXPECTED_JSPF = {
    "playlist": {
        "title": "Test Playlist",
        "track": [{"identifier": ["https://musicbrainz.org/recording/abc"]}]
    },
    "identifier": "some-playlist-mbid"
}


def test_import_from_xspf_calls_patch():
    mock_playlist = MagicMock()
    mock_playlist.get_jspf.return_value = {
        "playlist": {"title": "Test Playlist", "track": []}
    }
    mock_playlist.playlists = [MagicMock(mbid="some-playlist-mbid")]

    mock_patch_instance = MagicMock()
    mock_patch_instance.generate_playlist.return_value = mock_playlist

    with patch("listenbrainz.troi.import_ms.ImportXSPFPlaylistPatch",
               return_value=mock_patch_instance) as MockPatch:
        from listenbrainz.troi.import_ms import import_from_xspf
        result = import_from_xspf(SAMPLE_XSPF, "user-token")

    MockPatch.assert_called_once()
    args_passed = MockPatch.call_args[0][0]
    assert args_passed["xspf_content"] == SAMPLE_XSPF
    assert args_passed["token"] == "user-token"
    assert "identifier" in result
```

**Step 2: Run to confirm it fails**

```bash
cd /Users/ansh/Desktop/Ansh/gsoc/listenbrainz-server
python -m pytest listenbrainz/troi/tests/test_import_ms_xspf.py -v
```
Expected: `ImportError` or `AttributeError` — `import_from_xspf` does not exist yet.

**Step 3: Add `import_from_xspf` to `listenbrainz/troi/import_ms.py`**

Add at the bottom of the file:

```python
def import_from_xspf(xspf_content, user):
    from troi.patches.playlist_from_xspf import ImportXSPFPlaylistPatch
    args = {
        "xspf_content": xspf_content,
        "token": user,
        "upload": True,
        "created_for": None,
        "echo": False,
        "min_recordings": 1
    }
    patch = ImportXSPFPlaylistPatch(args)
    playlist = patch.generate_playlist()
    result = playlist.get_jspf()
    result.update({"identifier": playlist.playlists[0].mbid})
    return result
```

**Step 4: Run tests to confirm they pass**

```bash
python -m pytest listenbrainz/troi/tests/test_import_ms_xspf.py -v
```
Expected: 1 test PASS.

**Step 5: Commit**

```bash
git add listenbrainz/troi/import_ms.py listenbrainz/troi/tests/test_import_ms_xspf.py
git commit -m "feat: add import_from_xspf backend function"
```

---

### Task 5: Backend API endpoint

**Files:**
- Modify: `listenbrainz/webserver/views/playlist_api.py`
- Modify: `listenbrainz/tests/integration/test_playlist_api.py`

**Step 1: Write the failing integration test**

In `listenbrainz/tests/integration/test_playlist_api.py`, add a new test class at the bottom:

```python
class XSPFImportTestCase(PlaylistAPITestCase):

    SAMPLE_XSPF = """<?xml version="1.0" encoding="UTF-8"?>
<playlist version="1" xmlns="http://xspf.org/ns/0/">
  <title>Test XSPF Playlist</title>
  <annotation>A test playlist</annotation>
  <trackList>
    <track>
      <title>Blue (Da Ba Dee)</title>
      <creator>Eiffel 65</creator>
    </track>
  </trackList>
</playlist>"""

    MOCK_JSPF = {
        "playlist": {
            "title": "Test XSPF Playlist",
            "annotation": "A test playlist",
            "extension": {
                "https://musicbrainz.org/doc/jspf#playlist": {"public": True}
            },
            "track": [
                {"identifier": ["https://musicbrainz.org/recording/e8f9b188-f819-4e43-ab0f-4bd26ce9ff56"]}
            ]
        }
    }

    @mock.patch("listenbrainz.webserver.views.playlist_api.import_from_xspf")
    def test_import_xspf_tracks_success(self, mock_import):
        mock_import.return_value = self.MOCK_JSPF

        response = self.client.post(
            self.custom_url_for("playlist_api_v1.import_tracks_from_xspf_playlist"),
            data=self.SAMPLE_XSPF,
            content_type="text/xml",
            headers={"Authorization": "Token {}".format(self.user["auth_token"])}
        )
        self.assert200(response)
        data = response.json
        assert "playlist" in data
        assert data["playlist"]["title"] == "Test XSPF Playlist"
        mock_import.assert_called_once_with(self.SAMPLE_XSPF, self.user["auth_token"])

    def test_import_xspf_tracks_unauthenticated(self):
        response = self.client.post(
            self.custom_url_for("playlist_api_v1.import_tracks_from_xspf_playlist"),
            data=self.SAMPLE_XSPF,
            content_type="text/xml",
        )
        self.assert401(response)

    def test_import_xspf_tracks_missing_body(self):
        response = self.client.post(
            self.custom_url_for("playlist_api_v1.import_tracks_from_xspf_playlist"),
            content_type="text/xml",
            headers={"Authorization": "Token {}".format(self.user["auth_token"])}
        )
        self.assert400(response)
```

**Step 2: Run to confirm it fails**

```bash
python -m pytest listenbrainz/tests/integration/test_playlist_api.py::XSPFImportTestCase -v
```
Expected: `BuildError` — endpoint `playlist_api_v1.import_tracks_from_xspf_playlist` does not exist.

**Step 3: Update the import in `playlist_api.py`**

Change line 17 from:
```python
from listenbrainz.troi.import_ms import import_from_spotify, import_from_apple_music, import_from_soundcloud
```
To:
```python
from listenbrainz.troi.import_ms import import_from_spotify, import_from_apple_music, import_from_soundcloud, import_from_xspf
```

**Step 4: Add the endpoint to `playlist_api.py`**

After the SoundCloud import endpoint (after the function `import_tracks_from_soundcloud_playlist`, around line 1070), add:

```python
@playlist_api_bp.post("/xspf/tracks")
@crossdomain
@ratelimit()
@api_listenstore_needed
def import_tracks_from_xspf_playlist():
    """
    Import a playlist from an XSPF file and convert its tracks to JSPF.

    The request body must contain the raw XSPF XML content.
    Tracks are resolved to MusicBrainz recording MBIDs automatically:
    tracks with a <identifier> pointing to a MB recording URI are used
    directly; all others are resolved via the LB metadata lookup API.

    :reqheader Authorization: Token <user token>
    :reqheader Content-Type: text/xml
    :statuscode 200: tracks resolved and returned as JSPF.
    :statuscode 400: missing or empty XSPF content.
    :statuscode 401: invalid authorization.
    :resheader Content-Type: *application/json*
    """
    user = validate_auth_header()

    xspf_content = request.get_data(as_text=True)
    if not xspf_content:
        raise APIBadRequest("No XSPF content provided in request body.")

    try:
        playlist = import_from_xspf(xspf_content, user["auth_token"])
        return jsonify(playlist)
    except Exception as exc:
        raise APIInternalServerError(f"Could not import XSPF playlist: {str(exc)}")
```

**Step 5: Run tests to confirm they pass**

```bash
python -m pytest listenbrainz/tests/integration/test_playlist_api.py::XSPFImportTestCase -v
```
Expected: 3 tests PASS.

**Step 6: Run the full integration test suite to check for regressions**

```bash
python -m pytest listenbrainz/tests/integration/test_playlist_api.py -v --tb=short 2>&1 | tail -20
```
Expected: no regressions.

**Step 7: Commit**

```bash
git add listenbrainz/webserver/views/playlist_api.py \
        listenbrainz/tests/integration/test_playlist_api.py
git commit -m "feat: add POST /api/v1/playlist/xspf/tracks endpoint"
```

---

### Task 6: Frontend API service method

**Files:**
- Modify: `frontend/js/src/utils/APIService.ts`

**Step 1: Add `importXSPFPlaylistTracks` method**

Find `importSoundCloudPlaylistTracks` in `APIService.ts` (around line 1561). Add the new method immediately after it:

```typescript
importXSPFPlaylistTracks = async (
  userToken: string,
  xspfContent: string
): Promise<any> => {
  const url = `${this.APIBaseURI}/playlist/xspf/tracks`;
  const response = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Token ${userToken}`,
      "Content-Type": "text/xml",
    },
    body: xspfContent,
  });
  await this.checkStatus(response);
  return response.json();
};
```

**Step 2: Confirm TypeScript compiles cleanly**

```bash
cd /Users/ansh/Desktop/Ansh/gsoc/listenbrainz-server
yarn tsc --noEmit 2>&1 | grep -i error | head -20
```
Expected: no new errors.

**Step 3: Commit**

```bash
git add frontend/js/src/utils/APIService.ts
git commit -m "feat: add importXSPFPlaylistTracks to APIService"
```

---

### Task 7: ImportXSPFPlaylistModal React component

**Files:**
- Create: `frontend/js/src/user/playlists/components/ImportXSPFPlaylistModal.tsx`

**Step 1: Create the component**

```tsx
import * as React from "react";
import NiceModal, { useModal, bootstrapDialog } from "@ebay/nice-modal-react";
import { Modal } from "react-bootstrap";
import { toast } from "react-toastify";
import { Link } from "react-router";
import GlobalAppContext from "../../../utils/GlobalAppContext";
import { ToastMsg } from "../../../notifications/Notifications";

export default NiceModal.create(() => {
  const modal = useModal();
  const { currentUser, APIService } = React.useContext(GlobalAppContext);

  const [fileError, setFileError] = React.useState<string | null>(null);
  const [xspfContent, setXspfContent] = React.useState<string | null>(null);
  const [isLoading, setIsLoading] = React.useState(false);

  const importPlaylist = React.useCallback(async (): Promise<
    JSPFPlaylist | undefined
  > => {
    if (!currentUser?.auth_token) {
      toast.error(
        <ToastMsg
          title="Error"
          message="You must be logged in for this operation"
        />,
        { toastId: "auth-error" }
      );
      return undefined;
    }
    if (!xspfContent) return undefined;

    try {
      setIsLoading(true);

      // Step 1: Resolve XSPF tracks → JSPF via troi pipeline
      const jspfObject = await APIService.importXSPFPlaylistTracks(
        currentUser.auth_token,
        xspfContent
      );

      // Step 2: Create the playlist using the existing endpoint
      const newPlaylistId = await APIService.createPlaylist(
        currentUser.auth_token,
        jspfObject
      );

      toast.success(
        <ToastMsg
          title="Created playlist"
          message={
            <>
              Created a new playlist:{" "}
              <Link to={`/playlist/${newPlaylistId}`}>{newPlaylistId}</Link>
            </>
          }
        />,
        { toastId: "create-playlist-success" }
      );

      // Step 3: Fetch and return the playlist so the page can update
      const response = await APIService.getPlaylist(
        newPlaylistId,
        currentUser.auth_token
      );
      const fetched: JSPFObject = await response.json();
      return fetched.playlist;
    } catch (error) {
      toast.error(
        <ToastMsg
          title="Could not import XSPF playlist"
          message={`Something went wrong: ${error.toString()}`}
        />,
        { toastId: "import-xspf-error" }
      );
      return undefined;
    } finally {
      setIsLoading(false);
    }
  }, [currentUser, xspfContent, APIService]);

  const handleFileChange = (
    event: React.ChangeEvent<HTMLInputElement>
  ): void => {
    const selectedFile = event.target.files?.[0];
    setFileError(null);
    setXspfContent(null);

    if (!selectedFile) return;

    const fileExtension = selectedFile.name.split(".").pop()?.toLowerCase();
    if (fileExtension !== "xspf") {
      setFileError("Invalid file format. Please select a valid .xspf file.");
      return;
    }

    const reader = new FileReader();
    reader.onload = (e: ProgressEvent<FileReader>) => {
      setXspfContent(e.target?.result as string);
    };
    reader.onerror = () => {
      setFileError("Error reading file. Please try again.");
    };
    reader.readAsText(selectedFile);
  };

  const onSubmit = async (event: React.SyntheticEvent) => {
    try {
      const newPlaylist = await importPlaylist();
      if (!newPlaylist) return;
      modal.resolve(newPlaylist);
      modal.hide();
    } catch (error) {
      toast.error(
        <ToastMsg
          title="Something went wrong"
          message={<>We could not import your playlist: {error.toString()}</>}
        />,
        { toastId: "save-playlist-error" }
      );
    }
  };

  return (
    <Modal
      {...bootstrapDialog(modal)}
      title="Import XSPF playlist"
      aria-labelledby="ImportXSPFPlaylistModalLabel"
      id="ImportXSPFPlaylistModal"
    >
      <Modal.Header closeButton>
        <Modal.Title id="ImportXSPFPlaylistModalLabel">
          Import XSPF playlist
        </Modal.Title>
      </Modal.Header>
      <Modal.Body>
        <div>
          <label className="form-label" htmlFor="xspfPlaylistFile">
            Choose or drop a file with .xspf extension
          </label>
          <input
            type="file"
            className="form-control"
            id="xspfPlaylistFile"
            accept=".xspf"
            onChange={handleFileChange}
          />
        </div>
        {fileError && <div className="has-error">{fileError}</div>}
        <p className="form-text">
          XSPF is an open playlist format exported by tools like{" "}
          <a
            href="https://soundiiz.com"
            target="_blank"
            rel="noopener noreferrer"
          >
            Soundiiz
          </a>
          . Tracks will be resolved to MusicBrainz recordings automatically.
          Tracks that cannot be matched will be skipped.
        </p>
      </Modal.Body>
      <Modal.Footer>
        <button
          type="button"
          className="btn btn-secondary"
          onClick={modal.hide}
        >
          Cancel
        </button>
        <button
          type="submit"
          className="btn btn-primary"
          disabled={!currentUser?.auth_token || xspfContent === null || isLoading}
          onClick={onSubmit}
        >
          {isLoading ? "Importing…" : "Import"}
        </button>
      </Modal.Footer>
    </Modal>
  );
});
```

**Step 2: Confirm TypeScript compiles cleanly**

```bash
yarn tsc --noEmit 2>&1 | grep -i error | head -20
```
Expected: no new errors.

**Step 3: Commit**

```bash
git add frontend/js/src/user/playlists/components/ImportXSPFPlaylistModal.tsx
git commit -m "feat: add ImportXSPFPlaylistModal component"
```

---

### Task 8: Wire XSPF import into the Playlists page

**Files:**
- Modify: `frontend/js/src/user/playlists/Playlists.tsx`

**Step 1: Add import statement**

At the top of `Playlists.tsx`, after line 30 (`import ImportSoundCloudPlaylistModal ...`), add:

```typescript
import ImportXSPFPlaylistModal from "./components/ImportXSPFPlaylistModal";
```

**Step 2: Add dropdown button**

Find the "Upload JSPF file" button in the import dropdown (around line 418). Insert the XSPF button immediately **before** it:

```tsx
<button
  type="button"
  onClick={() => {
    NiceModal.show<JSPFPlaylist | JSPFPlaylist[], any>(
      ImportXSPFPlaylistModal
    ).then((playlist) => {
      if (Array.isArray(playlist)) {
        playlist.forEach((p: JSPFPlaylist) => {
          this.onPlaylistCreated(p);
        });
      } else {
        this.onPlaylistCreated(playlist);
      }
    });
  }}
  className="dropdown-item"
>
  <FontAwesomeIcon icon={faFileImport} />
  &nbsp;Upload XSPF file
</button>
```

**Step 3: Build to confirm no TypeScript errors**

```bash
cd /Users/ansh/Desktop/Ansh/gsoc/listenbrainz-server
yarn tsc --noEmit 2>&1 | grep -i error | head -20
```
Expected: no new errors.

**Step 4: Commit**

```bash
git add frontend/js/src/user/playlists/Playlists.tsx
git commit -m "feat: add XSPF import option to playlist import dropdown"
```

---

## Summary of all files changed

### troi-recommendation-playground
| Action | File |
|--------|------|
| Create | `troi/tools/xspf_lookup.py` |
| Create | `tests/test_xspf_lookup.py` |
| Modify | `troi/playlist.py` (add `RecordingsFromXSPFElement`) |
| Create | `tests/test_xspf_element.py` |
| Create | `troi/patches/playlist_from_xspf.py` |
| Create | `tests/test_playlist_from_xspf_patch.py` |

### listenbrainz-server
| Action | File |
|--------|------|
| Modify | `listenbrainz/troi/import_ms.py` |
| Create | `listenbrainz/troi/tests/test_import_ms_xspf.py` |
| Modify | `listenbrainz/webserver/views/playlist_api.py` |
| Modify | `listenbrainz/tests/integration/test_playlist_api.py` |
| Modify | `frontend/js/src/utils/APIService.ts` |
| Create | `frontend/js/src/user/playlists/components/ImportXSPFPlaylistModal.tsx` |
| Modify | `frontend/js/src/user/playlists/Playlists.tsx` |
