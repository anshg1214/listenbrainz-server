# Typed SQL Layer — Phase 2a: MsidMbidModel Family

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate `MsidMbidModel` and all its subclasses (`Feedback`, `PinnedRecording`, `WritablePinnedRecording`, and three models in `user_timeline_event.py`) from `pydantic.v1` to native Pydantic v2 syntax.

**Architecture:** These models share an inheritance chain — `MsidMbidModel` must be migrated first, then each subclass. All four model files change together in this PR because a v2 base class cannot have v1 subclasses. The key syntactic changes are: `validator` → `field_validator`, `root_validator` → `model_validator`, `NonNegativeInt` → `Field(ge=0)`, `constr(max_length=N)` → `Field(max_length=N)`, `conlist(T, min_items=N)` → `Annotated[list[T], Field(min_length=N)]`, `.dict()` → `.model_dump()`.

**Tech Stack:** Python 3.12, Pydantic 2.x (native — no `pydantic.v1`), SQLAlchemy 2.0

**Branch:** Create a new branch from `ansh/update-pydantic`.

**Spec:** `docs/superpowers/specs/2026-04-05-typed-sql-layer-design.md`

**Run tests with:**
```bash
docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
  run --rm listenbrainz pytest <test-file> -v
```
*(Requires Docker containers running: `./test.sh -u` to bring them up if needed)*

---

## File Map

| Action | File | Change |
|---|---|---|
| Modify | `listenbrainz/db/msid_mbid_mapping.py` | `MsidMbidModel`: `validator` → `field_validator`, `root_validator` → `model_validator` |
| Modify | `listenbrainz/db/model/feedback.py` | `Feedback`: `NonNegativeInt` → `Field(ge=0)`, `validator` → `field_validator`, `.dict()` → `.model_dump()` |
| Modify | `listenbrainz/db/model/pinned_recording.py` | `PinnedRecording`, `WritablePinnedRecording`: `constr` → `Field(max_length=...)`, `NonNegativeInt` → `Field(ge=0)`, complex validators rewritten |
| Modify | `listenbrainz/db/model/user_timeline_event.py` | Only the 3 `MsidMbidModel` subclasses: `conlist` → `Annotated[list, Field(min_length=1)]` |
| Modify | `listenbrainz/db/tests/test_pinned_recording.py` | `from pydantic.v1 import ValidationError` → `from pydantic import ValidationError` |
| Modify | `listenbrainz/db/tests/test_validators.py` | Same import fix |

---

## Task 1: Migrate `MsidMbidModel` to native Pydantic v2

**Files:**
- Modify: `listenbrainz/db/msid_mbid_mapping.py`
- Test: `listenbrainz/db/tests/test_msid_mbid_mapping.py` (no changes needed — uses `assertRaisesRegex(ValueError, ...)`, not `ValidationError`)

- [ ] **Step 1: Run the existing test to confirm it passes (baseline)**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_msid_mbid_mapping.py::MappingTestCase::test_msid_mbid_model -v
  ```

  Expected: PASS. If it fails, stop and investigate before continuing.

- [ ] **Step 2: Replace the model in `msid_mbid_mapping.py`**

  Replace the current `MsidMbidModel` and its imports. The full new version of the model section (lines 1–40 of the file):

  ```python
  from collections import defaultdict
  from typing import List, Optional, TypeVar, Annotated

  from flask import current_app
  from psycopg2.extras import DictCursor
  from pydantic import BaseModel, field_validator, model_validator, Field

  from data.model.validators import check_valid_uuid
  from listenbrainz.db.recording import load_recordings_from_mbids
  from listenbrainz.messybrainz import load_recordings_from_msids
  from sqlalchemy.engine import Connection


  class MsidMbidModel(BaseModel):
      """
      A model to use as base for tables which support both msid and mbids.

      We intend to migrate from msids to mbids but for compatibility and till
      the remaining gaps in mapping are filled we want to support msids as well.
      """
      recording_msid: Optional[str] = None
      recording_mbid: Optional[str] = None
      track_metadata: Optional[dict] = None

      @field_validator("recording_msid", "recording_mbid", mode="before")
      @classmethod
      def validate_uuid(cls, v):
          return check_valid_uuid(v)

      @model_validator(mode="after")
      def check_at_least_mbid_or_msid(self):
          if self.recording_msid is None and self.recording_mbid is None:
              raise ValueError("at least one of recording_msid or recording_mbid should be specified")
          return self


  ModelT = TypeVar('ModelT', bound=MsidMbidModel)
  ```

  The rest of the file (`fetch_track_metadata_for_items`, `_update_mbid_items`, `_update_msid_items`) stays unchanged.

- [ ] **Step 3: Run the test again — must pass**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_msid_mbid_mapping.py::MappingTestCase::test_msid_mbid_model -v
  ```

  Expected: PASS. The test uses `assertRaisesRegex(ValueError, ...)` which works with both v1 and v2 since Pydantic v2 wraps validation errors as `ValueError` when raised from validators.

  If the test fails with `pydantic.ValidationError` not being a `ValueError`:
  - In Pydantic v2, `ValidationError` is NOT a subclass of `ValueError`. The test catches `ValueError` at the point of construction (`MsidMbidModel(...)`). Update `test_msid_mbid_model` to catch `ValidationError` instead:

  ```python
  from pydantic import ValidationError

  def test_msid_mbid_model(self):
      with self.assertRaises(ValidationError):
          model = MsidMbidModel(recording_mbid=None, recording_msid=None)

      with self.assertRaises(ValidationError):
          model = MsidMbidModel(recording_msid='', recording_mbid=str(uuid.uuid4()))

      with self.assertRaises(ValidationError):
          model = MsidMbidModel(recording_msid=str(uuid.uuid4()), recording_mbid='')

      # test that 2 valid uuids doesn't raise error
      model = MsidMbidModel(recording_msid=str(uuid.uuid4()), recording_mbid=str(uuid.uuid4()))
  ```

- [ ] **Step 4: Commit**

  ```bash
  git add listenbrainz/db/msid_mbid_mapping.py listenbrainz/db/tests/test_msid_mbid_mapping.py
  git commit -m "refactor: migrate MsidMbidModel to native Pydantic v2"
  ```

---

## Task 2: Migrate `Feedback` model

**Files:**
- Modify: `listenbrainz/db/model/feedback.py`
- Test: `listenbrainz/db/tests/test_feedback.py` (check for ValidationError imports)

- [ ] **Step 1: Run existing feedback tests (baseline)**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_feedback.py -v
  ```

  Expected: PASS.

- [ ] **Step 2: Replace `feedback.py`**

  Full new content of `listenbrainz/db/model/feedback.py`:

  ```python
  from copy import copy

  from datetime import datetime
  from typing import Optional

  from pydantic import field_validator, Field

  from listenbrainz.db.msid_mbid_mapping import MsidMbidModel


  class Feedback(MsidMbidModel):
      """ Represents a feedback object
          Args:
              user_id: the row id of the user in the DB
              user_name: (Optional) the MusicBrainz ID of the user
              recording_msid: the MessyBrainz ID of the recording
              score: the score associated with the recording (+1/-1 for love/hate respectively)
              created: (Optional)the timestamp when the feedback record was inserted into DB
      """

      user_id: int = Field(ge=0)
      user_name: Optional[str] = None
      score: int
      created: Optional[datetime] = None

      def to_api(self) -> dict:
          data = self.model_dump()
          data["user_id"] = self.user_name
          if self.created is not None:
              data["created"] = int(self.created.timestamp())
          data.pop("user_name", None)
          return data

      @field_validator('score')
      @classmethod
      def check_score_is_valid(cls, scr):
          if scr not in [-1, 0, 1]:
              raise ValueError('Score can have a value of 1, 0 or -1.')
          return scr
  ```

- [ ] **Step 3: Check if `test_feedback.py` imports `pydantic.v1.ValidationError`**

  ```bash
  grep -n "pydantic" listenbrainz/db/tests/test_feedback.py
  ```

  If it has `from pydantic.v1 import ValidationError`, change it to `from pydantic import ValidationError`.

- [ ] **Step 4: Run feedback tests — must pass**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_feedback.py -v
  ```

  Expected: PASS.

- [ ] **Step 5: Commit**

  ```bash
  git add listenbrainz/db/model/feedback.py listenbrainz/db/tests/test_feedback.py
  git commit -m "refactor: migrate Feedback model to native Pydantic v2"
  ```

---

## Task 3: Migrate `PinnedRecording` and `WritablePinnedRecording`

**Files:**
- Modify: `listenbrainz/db/model/pinned_recording.py`
- Modify: `listenbrainz/db/tests/test_pinned_recording.py` (fix ValidationError import)

- [ ] **Step 1: Run existing pinned recording tests (baseline)**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_pinned_recording.py -v
  ```

  Expected: PASS.

- [ ] **Step 2: Update the ValidationError import in the test file**

  In `listenbrainz/db/tests/test_pinned_recording.py`, line 4:
  ```python
  # before
  from pydantic.v1 import ValidationError

  # after
  from pydantic import ValidationError
  ```

- [ ] **Step 3: Replace `pinned_recording.py`**

  Full new content of `listenbrainz/db/model/pinned_recording.py`:

  ```python
  from datetime import datetime, timedelta, timezone
  from typing import Optional, Annotated

  from pydantic import field_validator, model_validator, Field

  from data.model.validators import check_datetime_has_tzinfo
  from listenbrainz.db.msid_mbid_mapping import MsidMbidModel

  #: Default number of days after which a pinned recording expires (gets unpinned).
  DAYS_UNTIL_UNPIN = 7
  MAX_BLURB_CONTENT_LENGTH = 280  # maximum length of blurb content


  class PinnedRecording(MsidMbidModel):
      """Represents a pinned recording object.
      Args:
          user_id: the row id of the user in the DB
          user_name: (Optional) the name of the user associated with the user_id
          row_id: the row id of the pinned_recording in the DB
          recording_mbid: the MusicBrainz ID of the recording
          blurb_content: (Optional) the custom text content of the pinned recording
          created: the datetime containing tzinfo representing when the pinned recording record was inserted into DB
          pinned_until: the datetime containing tzinfo representing when the pinned recording is set to expire/unpin

          Validates that pinned_until contains tzinfo() and is greater than created.
      """

      user_id: int = Field(ge=0)
      user_name: Optional[str] = None
      row_id: int = Field(ge=0)
      blurb_content: Optional[str] = Field(default=None, max_length=MAX_BLURB_CONTENT_LENGTH)
      created: datetime
      pinned_until: datetime

      @field_validator("created", "pinned_until", mode="before")
      @classmethod
      def validate_tzinfo(cls, v):
          return check_datetime_has_tzinfo(v)

      @model_validator(mode="after")
      def check_pin_until_greater_than_created(self):
          try:
              if self.pinned_until <= self.created:
                  raise ValueError(
                      "Pinned_until of returned PinnedRecording must be greater than created."
                  )
          except (ValueError, AttributeError) as e:
              raise ValueError(str(e))
          return self

      def to_api(self):
          pin = self.model_dump()
          pin["created"] = int(pin["created"].timestamp())
          pin["pinned_until"] = int(pin["pinned_until"].timestamp())
          del pin["user_id"]
          if pin["user_name"] is None:
              del pin["user_name"]
          return pin


  class WritablePinnedRecording(PinnedRecording):
      """Represents a pinned recording object to pin/submit to the DB.
         This model does not require a row_id, initializes created to now(), and initializes pinned_until to now() + one week.

      Args:
          user_id: the row id of the user in the DB
          row_id: (Optional) the row id of the pinned_recording in the DB
          recording_mbid: the MusicBrainz ID of the recording
          blurb_content: (Optional) the custom text content of the pinned recording
          created: (Optional) the datetime containing tzinfo representing when the pinned recording record was inserted into DB
          pinned_until: (Optional) the datetime containing tzinfo representing when the pinned recording is set to expire/unpin

      """

      row_id: int = Field(default=None, ge=0)
      created: Optional[datetime] = None
      pinned_until: Optional[datetime] = None

      @model_validator(mode="before")
      @classmethod
      def set_defaults(cls, data):
          if isinstance(data, dict):
              if not data.get("created"):
                  data["created"] = datetime.now(timezone.utc)
              if not data.get("pinned_until"):
                  data["pinned_until"] = data["created"] + timedelta(days=DAYS_UNTIL_UNPIN)
          return data
  ```

- [ ] **Step 4: Run pinned recording tests — must pass**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_pinned_recording.py -v
  ```

  Expected: PASS. If any test fails with a `ValidationError` structure difference (v2 errors have a different `errors()` format than v1), check if the test inspects `exc.errors()` fields — update field names from v1 (`loc`, `msg`, `type`) to v2 (`loc`, `msg`, `type`, `input`, `url`). Only `loc` and `msg` are typically used in assertions.

- [ ] **Step 5: Commit**

  ```bash
  git add listenbrainz/db/model/pinned_recording.py listenbrainz/db/tests/test_pinned_recording.py
  git commit -m "refactor: migrate PinnedRecording and WritablePinnedRecording to native Pydantic v2"
  ```

---

## Task 4: Migrate MsidMbidModel subclasses in `user_timeline_event.py`

Only the three classes that inherit from `MsidMbidModel` need to change. All other classes in the file stay on `pydantic.v1` unchanged.

**Files:**
- Modify: `listenbrainz/db/model/user_timeline_event.py`
- Test: `listenbrainz/db/tests/test_user_timeline_event.py` (check for ValidationError imports)

- [ ] **Step 1: Run existing timeline event tests (baseline)**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_user_timeline_event.py -v
  ```

  Expected: PASS.

- [ ] **Step 2: Update the three MsidMbidModel subclasses in `user_timeline_event.py`**

  The file currently has:
  ```python
  from pydantic.v1 import BaseModel, NonNegativeInt, constr, conlist
  ```

  Add a new import line for native v2 (needed for `Field` and `Annotated`):
  ```python
  from typing import Annotated
  from pydantic import Field
  ```

  Then update only these three classes (leave all other classes in the file completely unchanged):

  **`RecordingRecommendationMetadata`** — no change needed, it's just `pass`:
  ```python
  class RecordingRecommendationMetadata(MsidMbidModel):
      pass
  ```

  **`WritePersonalRecordingRecommendationMetadata`** — change `conlist(str, min_items=1)` to native v2:
  ```python
  class WritePersonalRecordingRecommendationMetadata(MsidMbidModel):
      users: Annotated[list[str], Field(min_length=1)]
      blurb_content: Optional[str] = None
  ```

  **`PersonalRecordingRecommendationMetadata`** — already uses `list[str]`, no structural change needed but make `blurb_content` explicit:
  ```python
  class PersonalRecordingRecommendationMetadata(MsidMbidModel):
      users: list[str]
      blurb_content: Optional[str] = None
  ```

  Remove `conlist` from the `pydantic.v1` import line since it's no longer used:
  ```python
  # before
  from pydantic.v1 import BaseModel, NonNegativeInt, constr, conlist
  # after
  from pydantic.v1 import BaseModel, NonNegativeInt, constr
  ```

- [ ] **Step 3: Check for ValidationError import in test file**

  ```bash
  grep -n "pydantic" listenbrainz/db/tests/test_user_timeline_event.py
  ```

  If `from pydantic.v1 import ValidationError` is present, change it to `from pydantic import ValidationError`. Leave any other `pydantic.v1` imports alone.

- [ ] **Step 4: Run timeline event tests — must pass**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_user_timeline_event.py -v
  ```

  Expected: PASS.

- [ ] **Step 5: Also run test_validators.py**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/test_validators.py -v
  ```

  If `test_validators.py` has `from pydantic.v1 import ValidationError`, change it to `from pydantic import ValidationError` and re-run.

- [ ] **Step 6: Run the full db test suite to confirm no regressions**

  ```bash
  docker compose -f docker/docker-compose.test.yml -p listenbrainz_test \
    run --rm listenbrainz pytest listenbrainz/db/tests/ -v
  ```

  Expected: all tests pass.

- [ ] **Step 7: Commit**

  ```bash
  git add listenbrainz/db/model/user_timeline_event.py \
          listenbrainz/db/tests/test_user_timeline_event.py \
          listenbrainz/db/tests/test_validators.py
  git commit -m "refactor: migrate MsidMbidModel subclasses in user_timeline_event to native Pydantic v2"
  ```

---

## Self-Review Against Spec

| Spec requirement | Covered by |
|---|---|
| Migrate MsidMbidModel to native v2 | Task 1 |
| Migrate Feedback model | Task 2 |
| Migrate PinnedRecording + WritablePinnedRecording | Task 3 |
| Migrate MsidMbidModel subclasses in user_timeline_event | Task 4 |
| `validator` → `field_validator` | Tasks 1–4 |
| `root_validator` → `model_validator` | Tasks 1, 3 |
| `.dict()` → `.model_dump()` | Tasks 2, 3 |
| `NonNegativeInt` → `Field(ge=0)` | Tasks 2, 3 |
| `constr(max_length=N)` → `Field(max_length=N)` | Task 3 |
| `conlist(T, min_items=N)` → `Annotated[list[T], Field(min_length=N)]` | Task 4 |

**What this plan does NOT cover** (Phase 2b+):
- `user.py` — returns raw dicts to many callers, requires updating all call sites
- Remaining `listenbrainz/db/model/` files not in the MsidMbidModel family
- `data/model/` files
