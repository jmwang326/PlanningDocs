# Level 13 - Data Structures

## Purpose
Python class definitions for core data types. Used by L12 function signatures.

---

## Detection

**What it is:** Single YOLO inference result for one frame, one bounding box.

```python
@dataclass
class Detection:
    """Single detection from YOLO inference."""

    frame_time: float  # UTC timestamp
    bbox: BoundingBox
    confidence: float  # YOLO detection confidence [0.0, 1.0]
    class_name: str  # "person" | "vehicle" | "animal"
    cell_6x6: Tuple[int, int]  # Grid cell (x, y) containing bbox centroid
```

---

## BoundingBox

```python
@dataclass
class BoundingBox:
    """Bounding box in pixel coordinates."""

    x: int  # Top-left corner x
    y: int  # Top-left corner y
    width: int
    height: int

    @property
    def center(self) -> Tuple[int, int]:
        """Returns (center_x, center_y)."""
        return (self.x + self.width // 2, self.y + self.height // 2)

    @property
    def area(self) -> int:
        """Returns bbox area in pixels."""
        return self.width * self.height
```

---

## LocalTrack

**What it is:** One observation from YOLO tracking in one 12-second chunk on one camera.

```python
@dataclass
class LocalTrack:
    """Single track observation from one chunk."""

    # Primary key (database)
    id: Optional[int] = None  # None until saved to database

    # Identity within chunk
    local_id: int  # YOLO tracking ID (chunk-scoped, resets each chunk)
    camera_id: str
    chunk_id: int

    # Timing
    start_time: float  # UTC timestamp
    end_time: float  # UTC timestamp (variable duration, up to 12s)

    # Spatial (6x6 grid)
    entry_cell: Tuple[int, int]  # (x, y) where track first appears
    exit_cell: Tuple[int, int]  # (x, y) where track last appears

    # Observations
    detections: List[Detection]  # Frame-by-frame detections

    # Relationships (track IDs, None if unlinked)
    previous_track_id: Optional[int]  # Same camera, previous chunk
    next_track_id: Optional[int]  # Same camera, next chunk

    # Identity (resolved or uncertain)
    global_agent_id: Optional[int]  # None if uncertain
    identity_candidates: Set[int]  # Empty if resolved

    # Context flags
    inside_vehicle: bool = False
    at_portal: Optional[str] = None  # Portal ID if started/ended at portal
    exit_reason: str = "left_frame"  # "left_frame" | "chunk_boundary" | "portal_crossed"

    # Computed properties (not stored in database)
    @property
    def duration(self) -> float:
        """Track duration in seconds."""
        return self.end_time - self.start_time

    @property
    def bbox_count(self) -> int:
        """Number of detections (frames)."""
        return len(self.detections)

    @property
    def is_resolved(self) -> bool:
        """True if identity is known."""
        return self.global_agent_id is not None

    @property
    def is_uncertain(self) -> bool:
        """True if identity is uncertain."""
        return self.global_agent_id is None and len(self.identity_candidates) > 0
```

**Note:** `merged_tracks` relationship not included here (NEEDS SPEC in L13_Database)

---

## Frame

**What it is:** Single JPEG snapshot with metadata from Blue Iris.

```python
@dataclass
class Frame:
    """Single frame from camera."""

    camera_id: str
    timestamp: float  # UTC timestamp
    profile: str  # "sub" | "main"
    jpeg_bytes: bytes  # Raw JPEG data

    # Metadata (optional, from Blue Iris if available)
    width: Optional[int] = None
    height: Optional[int] = None
```

---

## Evidence

**What it is:** Evidence for resolving uncertain tracks (face match, human review, LLM decision).

```python
@dataclass
class Evidence:
    """Evidence for identity resolution."""

    track_id: int  # Which track to resolve
    source: str  # "face_recognition" | "human_review" | "llm"

    # Decision
    decision: str  # "merge" | "reject" | "uncertain"
    agent_id: Optional[int] = None  # Which agent (if merge decision)

    # Metadata
    confidence: Optional[float] = None  # Face recognition confidence
    reasoning: Optional[str] = None  # LLM/human explanation
    timestamp: float = None  # When evidence collected
```

---

## CameraState

**What it is:** Current state of camera (for GPU allocation and chunk processing).

```python
@dataclass
class CameraState:
    """Camera state machine."""

    camera_id: str
    current_state: str  # "Standby" | "Armed" | "Active" | "Post"
    state_entered_at: float  # UTC timestamp when entered this state
    last_chunk_end_time: Optional[float]  # When last chunk was processed
```

---

## GridTransition

**What it is:** Single cell-to-cell transition in 6×6 grid (for grid learning queries).

```python
@dataclass
class GridTransition:
    """Learned travel time between camera grid cells."""

    from_camera: str
    to_camera: str
    from_cell: Tuple[int, int]  # (x, y)
    to_cell: Tuple[int, int]  # (x, y)
    agent_class: str  # "person" | "vehicle"
    min_travel_time: float  # Minimum observed travel time (seconds)

    @property
    def is_learned(self) -> bool:
        """True if path has been learned (< 1000s)."""
        return self.min_travel_time < 1000.0

    @property
    def is_overlap(self) -> bool:
        """True if cameras have overlapping FOV (< 0.4s)."""
        return self.min_travel_time < 0.4
```

---

## FaceCrop

**What it is:** Face crop extracted from track for recognition library.

```python
@dataclass
class FaceCrop:
    """Face crop for recognition library."""

    id: Optional[int] = None  # Database ID
    track_id: int
    global_agent_id: Optional[int]  # None if uncertain

    file_path: str  # e.g., "/data/faces/agent_123/cam_A_12345.jpg"
    timestamp: float
    camera_id: str

    validated: bool = False  # Human validated for training
    in_library: bool = False  # Currently in CodeProject.AI library
```

---

## Notes

- **All dataclasses use Python 3.7+ `@dataclass` decorator**
- **Optional fields:** Use `Optional[T]` for nullable fields
- **Timestamps:** Always float (UTC seconds since epoch)
- **Tuples for grid cells:** `(x, y)` where x, y ∈ [0, 5]
- **Database IDs:** `Optional[int]` until saved (None = not persisted yet)
- **Computed properties:** Use `@property` for derived fields (not stored)

---

## Type Aliases

```python
CameraID = str
ChunkID = int
AgentID = int
TrackID = int
GridCell = Tuple[int, int]  # (x, y) where 0 <= x, y <= 5
Timestamp = float  # UTC seconds since epoch
```

**Usage in L12 function signatures:**
```python
def process_chunk(camera_id: CameraID, frames: List[Frame]) -> List[LocalTrack]:
    ...
```
