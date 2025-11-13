# Data & Storage Architecture — Complete Pre-Planning

**Purpose:** Comprehensive pre-planning of all data structures, storage formats, functions, and libraries needed across all phases. Ensures we capture everything from Phase 0 that later phases will need.

**Philosophy:** Position data structures early, even if not used until later phases. Bootstrap with placeholders, fill in as we go. Avoid "we should have logged that from the start" regrets.

---

## Table of Contents

1. [Data Categories](#data-categories)
2. [Directory Structure](#directory-structure)
3. [Learned Data (Persistent State)](#learned-data-persistent-state)
4. [Raw Data (Frames & Media)](#raw-data-frames--media)
5. [Metadata (Tracks, Events, Decisions)](#metadata-tracks-events-decisions)
6. [Configuration Files](#configuration-files)
7. [Database Schema](#database-schema)
8. [File Formats & Serialization](#file-formats--serialization)
9. [Function Inventory by Phase](#function-inventory-by-phase)
10. [Library Dependencies](#library-dependencies)
11. [Configuration UI Tools](#configuration-ui-tools)
12. [Data Migration & Versioning](#data-migration--versioning)

---

## Data Categories

### 1. **Learned Data** (Persistent State)
Data learned from observations that improves over time. Must persist across restarts.

- **Height calibration grid (32×32 per camera):** Expected person bbox height per cell
- **Portal timing statistics:** Actual travel times through configured portals (for window tightening)
- **Face embeddings library:** Known people with face vectors
- **Re-ID fingerprints (Phase 4+):** Visual embeddings per agent per camera
- **Camera-specific parameters:** Grid overlays, ignore zones, perspective correction

### 2. **Raw Data** (Frames & Media)
Actual image/video content. Large, pruned by retention policy.

- **JPEG frames (sub profile):** 10 FPS continuous from all cameras
- **JPEG frames (main profile):** High-res during Active state only
- **MP4 clips (optional):** Reconstructed from frames for easy playback
- **Face crops:** Extracted faces for recognition/library building
- **Panel images:** 2×6 comparison panels for merge review

### 3. **Metadata** (Tracks, Events, Decisions)
Structured logs of what happened, when, and why. Critical for audit/debugging/learning.

- **Detection logs:** YOLO outputs per frame (sampled, not all frames)
- **Track segments:** Agent tracks with bbox sequences, camera, timing
- **Merge decisions:** Candidates considered, evidence, decision, outcome
- **Review queue:** Pending human/LLM decisions
- **Validation records:** Human/LLM corrections (ground truth for learning)
- **Event manifests:** Frame lists for reconstruction
- **State transitions:** Camera state changes with timestamps

### 4. **Configuration** (User-Editable)
System behavior defined by user. Version-controlled, hot-reloadable.

- **Stage config:** DEV_SLOW, DEV_FULL, PROD_LEARNING, PROD_AUTO
- **Camera definitions:** Names, BlueIris shortnames, ignore zones
- **Portal configuration:** Camera connections, cells, time windows
- **Thresholds:** Confidence levels, timing windows, evidence weights
- **External services:** CodeProject.AI endpoints, LLM API keys
- **Retention policies:** How long to keep frames, metadata, learned data

---

## Directory Structure

```
c:/marengo/
├── config/
│   ├── stages/
│   │   ├── DEV_SLOW.yaml
│   │   ├── DEV_FULL.yaml
│   │   ├── PROD_LEARNING.yaml
│   │   └── PROD_AUTO.yaml
│   ├── cameras.yaml              # Camera definitions, BlueIris shortnames, ignore zones
│   ├── portals.yaml               # Portal configuration (user-editable via UI)
│   ├── thresholds.yaml            # Confidence, timing, evidence thresholds
│   ├── services.yaml              # External AI services (CodeProject.AI, LLM)
│   ├── retention.yaml             # Data retention policies
│   └── secrets.encrypted          # Encrypted API keys, credentials
│
├── state/                         # Learned data (persistent across restarts)
│   ├── height_grids/
│   │   ├── FrontDoor_height_32x32.json
│   │   ├── Driveway_height_32x32.json
│   │   └── ...
│   ├── portal_stats/
│   │   └── portal_timing.json     # Learned travel times per portal
│   ├── face_library/
│   │   ├── embeddings.db          # SQLite: person_id, embedding_vector, metadata
│   │   └── crops/                 # Face image files
│   │       ├── person_001/
│   │       │   ├── 20251107_140523_001.jpg
│   │       │   └── ...
│   │       └── ...
│   ├── reid_fingerprints/         # Phase 4+ (deferred)
│   │   └── reid.db                # SQLite: agent_id, camera_id, embedding, metadata
│   ├── camera_params/
│   │   ├── FrontDoor_grid.json    # Grid overlay parameters
│   │   ├── FrontDoor_ignore.json  # Ignore zones
│   │   └── ...
│   └── backups/                   # Weekly snapshots
│       ├── 20251107_state.tar.gz
│       └── ...
│
├── storage/                       # Raw frames and media
│   ├── frames/
│   │   ├── FrontDoor/
│   │   │   ├── 20251107/
│   │   │   │   ├── 14/            # Hour bucket
│   │   │   │   │   ├── 20251107_140500_000_sub.jpg
│   │   │   │   │   ├── 20251107_140500_100_sub.jpg  # 10 FPS = 100ms spacing
│   │   │   │   │   ├── 20251107_140501_200_main.jpg # Main during Active
│   │   │   │   │   └── ...
│   │   │   │   └── ...
│   │   │   └── ...
│   │   └── ...
│   ├── clips/                     # Reconstructed MP4s (optional)
│   │   ├── event_001/
│   │   │   ├── clip.mp4
│   │   │   └── manifest.json
│   │   └── ...
│   ├── faces/                     # Extracted face crops
│   │   ├── FrontDoor/
│   │   │   ├── 20251107/
│   │   │   │   ├── track_042/
│   │   │   │   │   ├── 20251107_140523_001_face.jpg
│   │   │   │   │   ├── 20251107_140524_500_face.jpg
│   │   │   │   │   └── panel.jpg  # 2×6 composite
│   │   │   │   └── ...
│   │   │   └── ...
│   │   └── ...
│   └── panels/                    # Merge decision panels
│       ├── merge_12345/
│       │   ├── panel_A.jpg        # Source track
│       │   ├── panel_B.jpg        # Destination track
│       │   └── decision.json      # Evidence, outcome, LLM response
│       └── ...
│
├── metadata/                      # Structured logs (SQLite + JSON)
│   ├── marengo.db                 # Main SQLite database
│   ├── detections/                # Per-frame detection logs (sampled)
│   │   ├── 20251107/
│   │   │   └── detections_14.jsonl  # JSONL for easy append
│   │   └── ...
│   ├── tracks/                    # Track segment logs
│   │   ├── 20251107/
│   │   │   └── tracks_14.jsonl
│   │   └── ...
│   ├── merges/                    # Merge decision logs
│   │   ├── 20251107/
│   │   │   └── merges_14.jsonl
│   │   └── ...
│   ├── validations/               # Human/LLM corrections (ground truth)
│   │   ├── 20251107/
│   │   │   └── validations.jsonl
│   │   └── ...
│   └── events/                    # Event manifests
│       ├── event_001_manifest.json
│       └── ...
│
├── logs/                          # System logs (not surveillance data)
│   ├── system.log
│   ├── errors.log
│   └── performance.log
│
└── tools/                         # Configuration UI scripts
    ├── config_portal_editor.py    # Portal configuration UI
    ├── config_camera_grid.py      # Grid overlay & ignore zones
    ├── config_threshold_tuner.py  # Threshold adjustment UI
    ├── config_validator.py        # YAML validation & hot-reload
    └── config_backup.py           # State backup & restore
```

---

## Learned Data (Persistent State)

### 1. Height Calibration Grid (32×32 per camera)

**Purpose:** Learn expected person bbox height per cell to normalize for depth/perspective.

**File:** `state/height_grids/{camera}_height_32x32.json`

**Schema:**
```json
{
  "camera": "FrontDoor",
  "grid_size": [32, 32],
  "calibrated_count": 847,      // Total observations used
  "last_updated": "2025-11-07T14:30:00Z",
  "cells": [
    {
      "row": 0, "col": 0,
      "mean_height_px": 0,       // Not calibrated yet (no observations)
      "std_height_px": 0,
      "observation_count": 0,
      "confidence": 0.0          // 0.0 = uncalibrated, 1.0 = high confidence
    },
    {
      "row": 15, "col": 20,
      "mean_height_px": 180.5,
      "std_height_px": 12.3,
      "observation_count": 42,
      "confidence": 0.85
    },
    // ... 1024 cells total (32×32)
  ],
  "version": "1.0"
}
```

**Update logic:**
- Online learning: Weighted average of existing mean + new observation
- Alpha = 0.2 (slower than portal timing, height is noisier)
- Only update if detection confidence > 0.6 and bbox not occluded
- Confidence grows with observation_count (saturates at ~50 observations)

**Functions needed:**
```python
def load_height_grid(camera: str) -> HeightGrid
def save_height_grid(camera: str, grid: HeightGrid)
def update_cell_height(grid: HeightGrid, row: int, col: int, 
                       observed_height: float, alpha: float = 0.2)
def get_expected_height(grid: HeightGrid, row: int, col: int) -> Optional[float]
def is_cell_calibrated(grid: HeightGrid, row: int, col: int, 
                       min_confidence: float = 0.5) -> bool
```

---

### 2. Portal Timing Statistics

**Purpose:** Learn actual travel times through configured portals to tighten time windows.

**File:** `state/portal_stats/portal_timing.json`

**Schema:**
```json
{
  "last_updated": "2025-11-07T14:30:00Z",
  "portals": [
    {
      "portal_id": "FrontDoor_to_Driveway",
      "camera_a": "FrontDoor",
      "camera_b": "Driveway",
      "cell_a": [0, 15],
      "cell_b": [0, 16],
      "direction": "bidirectional",
      
      "configured_typical_time": 2.0,    // User-configured (YAML)
      "configured_max_time": 10.0,
      
      "learned_stats": {
        "observation_count": 47,
        "mean_time": 2.3,
        "std_time": 0.8,
        "min_time": 1.1,
        "p50_time": 2.1,
        "p90_time": 3.5,
        "max_time": 5.2
      },
      
      "tightened_window": {
        "typical_time": 2.1,             // Learned P50
        "max_time": 4.0,                 // Learned P90 + margin
        "last_tightened": "2025-11-07T10:00:00Z",
        "confidence": 0.75               // Based on observation_count
      },
      
      "observations": [                   // Last 100 observations (rolling)
        {"timestamp": "2025-11-07T14:25:00Z", "delta_t": 2.1, "validated": true},
        {"timestamp": "2025-11-07T13:10:00Z", "delta_t": 2.5, "validated": true},
        // ... up to 100 recent
      ]
    }
    // ... all portals
  ],
  "version": "1.0"
}
```

**Update logic:**
- Only update from validated merges (human/LLM approved = ground truth)
- Keep rolling window of last 100 observations per portal
- Recompute stats (mean, P50, P90) after each validation
- Tighten window conservatively: `max_time = P90 + 2*std` (Phase 4)
- Never tighten below configured minimums (safety)

**Functions needed:**
```python
def load_portal_stats() -> PortalStats
def save_portal_stats(stats: PortalStats)
def record_portal_observation(portal_id: str, delta_t: float, validated: bool)
def get_portal_window(portal_id: str) -> Tuple[float, float]  # (typical, max)
def tighten_portal_windows(min_observations: int = 50)  # Phase 4
def should_use_learned_window(portal_id: str, min_confidence: float = 0.7) -> bool
```

---

### 3. Identity Registry (People & Vehicles)

**Purpose:** Central master list of all named entities - both people and vehicles. Single source of truth prevents duplicate identities across face library and Re-ID database.

**File:** `state/face_library/embeddings.db` (SQLite)

**Schema:**
```sql
-- Master identity registry (shared by face library and Re-ID)
CREATE TABLE identities (
    identity_id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,             -- "John Doe", "Honda Accord Blue", "Unknown_042"
    type TEXT NOT NULL,                    -- "person" or "vehicle"
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen TIMESTAMP,
    notes TEXT,                            -- "Delivery driver", "Neighbor's car"
    
    -- Statistics
    total_tracks INTEGER DEFAULT 0,        -- How many times tracked
    total_merges INTEGER DEFAULT 0,        -- How many validated merges
    first_appearance TIMESTAMP,
    
    -- Embedding counts
    face_count INTEGER DEFAULT 0,          -- How many face embeddings (persons only)
    reid_count INTEGER DEFAULT 0           -- How many Re-ID fingerprints (Phase 4+)
);

CREATE INDEX idx_identities_name ON identities(name);
CREATE INDEX idx_identities_type ON identities(type);
CREATE INDEX idx_identities_last_seen ON identities(last_seen DESC);

-- Face embeddings reference identities
CREATE TABLE face_embeddings (
    embedding_id INTEGER PRIMARY KEY AUTOINCREMENT,
    identity_id INTEGER NOT NULL,         -- FK to identities table
    embedding BLOB NOT NULL,              -- 512-dim float32 vector (2048 bytes)
    quality REAL NOT NULL,                -- Quality score 0.0-1.0
    source_track_id TEXT,                 -- Track that generated this
    source_camera TEXT,
    timestamp TIMESTAMP,
    image_path TEXT,                      -- Path to face crop
    FOREIGN KEY (identity_id) REFERENCES identities(identity_id)
);

CREATE INDEX idx_face_identity ON face_embeddings(identity_id);
CREATE INDEX idx_face_quality ON face_embeddings(quality DESC);
```

**Face crop files:** `state/face_library/crops/identity_{id:04d}/{timestamp}_{seq:03d}.jpg`

**Functions needed:**
```python
# Identity management (critical for preventing duplicates)
def get_identity_suggestions(entity_type: str = "person", limit: int = 10) -> List[dict]
def create_or_get_identity(name: str, entity_type: str) -> int  # Normalized lookup
def merge_identities(keep_id: int, duplicate_id: int)  # Admin cleanup tool
def rename_identity(identity_id: int, new_name: str)

# Face library operations
def add_face_embedding(identity_id: int, embedding: np.ndarray, 
                       quality: float, metadata: dict) -> int
def get_identity_embeddings(identity_id: int, min_quality: float = 0.5) -> List[np.ndarray]
def match_face(embedding: np.ndarray, threshold: float = 0.60) -> Optional[Tuple[int, float]]
def prune_low_quality_faces(identity_id: int, keep_top_k: int = 10)
def export_library() -> dict  # For backup
def import_library(data: dict)  # From backup
```

---

### 4. Camera Parameters

**Purpose:** Per-camera ignore zones and resolution info. Grid is now fixed 32×32, no flexible parameters.

**File:** `state/camera_params/{camera}_params.json`

**Schema:**
```json
{
  "camera": "FrontDoor",
  "grid_size": [32, 32],             // Fixed uniform grid
  "ignore_zones": [
    {
      "name": "tree_sway",
      "polygon": [[0.1, 0.2], [0.15, 0.2], [0.15, 0.3], [0.1, 0.3]],  // Normalized 0-1
      "reason": "Wind movement causes false detections"
    }
  ],
  "resolution": [1920, 1080],        // Native camera resolution
  "last_updated": "2025-11-07T14:00:00Z",
  "version": "1.0"
}
```

**Functions needed:**
```python
def load_camera_params(camera: str) -> CameraParams
def save_camera_params(camera: str, params: CameraParams)
def add_ignore_zone(camera: str, name: str, polygon: List[Tuple[float, float]])
def remove_ignore_zone(camera: str, name: str)
def is_point_in_ignore_zone(camera: str, x: float, y: float) -> bool
def get_cell_from_point(x: int, y: int, resolution: Tuple[int, int]) -> Tuple[int, int]
```

---

## Raw Data (Frames & Media)

### 1. JPEG Frames

**Purpose:** Continuous frame capture for reconstruction and analysis.

**File naming:** `storage/frames/{camera}/{YYYYMMDD}/{HH}/{YYYYMMDD}_{HHMMSS}_{mmm}_{profile}.jpg`

Example: `storage/frames/FrontDoor/20251107/14/20251107_140523_500_sub.jpg`

**Metadata embedded in filename:**
- `YYYYMMDD`: Date
- `HHMMSS`: Time (hour, minute, second)
- `mmm`: Milliseconds (000-900 in 100ms steps for 10 FPS)
- `profile`: `sub` (always) or `main` (Active state only)

**Retention policy (configurable):**
- **Armed/Active/Post frames:** Keep for 30 days (configurable)
- **Standby frames:** Keep for 7 days (configurable)
- **Frames referenced in events:** Keep indefinitely (or until event pruned)
- **Pruning:** Nightly job deletes frames older than retention period

**Functions needed:**
```python
def save_frame(camera: str, timestamp: datetime, profile: str, jpeg_bytes: bytes) -> str
def get_frame_path(camera: str, timestamp: datetime, profile: str) -> str
def frame_exists(path: str) -> bool
def load_frame(path: str) -> bytes
def prune_old_frames(older_than_days: int)
def get_frame_count(camera: str, date: str) -> int
def list_frames(camera: str, start: datetime, end: datetime) -> List[str]
```

---

### 2. Face Crops

**Purpose:** Extracted faces for recognition and library building.

**Path:** `storage/faces/{camera}/{YYYYMMDD}/track_{id:03d}/{timestamp}_{seq:03d}_face.jpg`

**Panel path:** `storage/faces/{camera}/{YYYYMMDD}/track_{id:03d}/panel.jpg`

**Retention:** Keep for 90 days, or indefinitely if face added to library.

**Functions needed:**
```python
def save_face_crop(camera: str, track_id: int, timestamp: datetime, 
                   crop: np.ndarray, quality: float) -> str
def save_face_panel(camera: str, track_id: int, crops: List[np.ndarray]) -> str
def load_face_crop(path: str) -> np.ndarray
def prune_old_face_crops(older_than_days: int = 90)
```

---

### 3. Merge Decision Panels

**Purpose:** 2×6 comparison panels for human/LLM review.

**Evidence source:** All evidence is computed from track database records (timing, portal match, face similarity). The panel generation function packages this evidence with visual frames for display - it does NOT compute new evidence.

**Path:** `storage/panels/merge_{id:06d}/panel_{A|B}.jpg`

**Decision metadata:** `storage/panels/merge_{id:06d}/decision.json`

**Schema:**
```json
{
  "merge_id": 12345,
  "timestamp": "2025-11-07T14:30:00Z",
  "source_track": {
    "track_id": 42,
    "camera": "FrontDoor",
    "end_time": "2025-11-07T14:29:50Z",
    "last_cell": [0, 15]
  },
  "dest_track": {
    "track_id": 101,
    "camera": "Driveway",
    "start_time": "2025-11-07T14:29:52Z",
    "first_cell": [0, 16]
  },
  "evidence": {
    "portal_match": true,
    "delta_t": 2.1,
    "face_similarity": 0.72,
    "strong_evidence_count": 2,
    "weak_evidence_count": 1
  },
  "decision": "REVIEW_LLM",  // AUTO_MERGE, REVIEW_LLM, REVIEW_HUMAN, REJECT
  "llm_response": {
    "answer": "YES",
    "confidence": 0.85,
    "reasons": "Same clothing, face match, plausible timing"
  },
  "human_validation": {
    "validated_by": "user",
    "outcome": "ACCEPT",      // ACCEPT, REJECT, DEFER
    "timestamp": "2025-11-07T15:00:00Z",
    "notes": "Definitely same person"
  },
  "panel_a_path": "storage/panels/merge_012345/panel_A.jpg",
  "panel_b_path": "storage/panels/merge_012345/panel_B.jpg"
}
```

**Retention:** Keep indefinitely (audit trail).

**Functions needed:**
```python
def create_merge_panel(source_track: Track, dest_track: Track, 
                       evidence: dict) -> int  # Returns merge_id
                       # NOTE: evidence dict already computed from tracks
def save_panel_images(merge_id: int, panel_a: np.ndarray, panel_b: np.ndarray)
def record_llm_decision(merge_id: int, response: dict)
def record_human_validation(merge_id: int, outcome: str, notes: str = None)
def get_merge_decision(merge_id: int) -> dict
def list_pending_reviews(decision_type: str = None) -> List[int]
```

---

## Metadata (Tracks, Events, Decisions)

### Agent-Track Relationship

**Key concept:** Agent (1) → Tracks (many)

- **Agent:** Persistent entity (person, vehicle) that exists across cameras and time
- **Track:** Observable window where we can SEE the agent on ONE camera
- One agent has many tracks (e.g., Person #7 seen on FrontDoor, then Driveway, then Garage)
- One track belongs to exactly one agent (enforced by foreign key)

**Example flow:**
1. Detection on FrontDoor → create Agent #7, create Track #42 (agent_id=7, camera=FrontDoor)
2. Track ends (person leaves frame)
3. Detection on Driveway → merge decision determines it's same person → create Track #101 (agent_id=7, camera=Driveway)
4. Now Agent #7 has two tracks: [Track #42 on FrontDoor, Track #101 on Driveway]

**Query patterns:**
```sql
-- Get all tracks for an agent (chronological movement)
SELECT * FROM tracks WHERE agent_id = 7 ORDER BY start_time;

-- Get agent for a track
SELECT agent_id FROM tracks WHERE track_id = 42;

-- Get agent's current location (most recent active track)
SELECT camera, last_cell_row, last_cell_col 
FROM tracks 
WHERE agent_id = 7 AND status = 'active' 
ORDER BY start_time DESC 
LIMIT 1;
```

**ID-swap handling:**
A track with `possibly_switched=1` may point to the wrong agent. This happens when:
- Two people cross paths and YOLO bbox jumps between them
- Spatial continuity check detects impossible movement
- Motion vector flips >90° without explanation

When swap detected:
1. Mark original track as `possibly_switched=1`
2. End original track at swap point
3. Create new track with new agent_id for the "other" person
4. Human review can later correct agent_id assignments if needed

**Multiple agents in one track (NOT SUPPORTED):**
A track can only belong to one agent at a time. If detection finds multiple people in frame:
- Each person gets their own track (separate track_id, separate agent_id)
- Tracks are associated via IoU frame-to-frame
- If two tracks collide/cross, mark both as `possibly_switched` and create new tracks

This is simpler than trying to track "which pixels belong to which agent" - we just acknowledge uncertainty and let human review fix it later.

---

### Multi-Scale Grid System & Merge Complexity

**Dual-scale grid rationale:**

**32×32 fine grid (intra-camera, stored in tracks table):**
- Purpose: Track agents within one camera, discriminate close-proximity agents
- Spatial resolution: ~60-80 pixels per cell (for 1920×1080 cameras)
- **Critical for multi-person scenarios:**
  - Two people walking side-by-side appear in different fine grid cells
  - Example: Both in coarse cell B3, but Person-Right at (5,12), Person-Left at (2,12)
  - Fine grid spatial consistency validates Re-ID matches across cameras
  - If person jumps from right→left between cameras → suspicious, flag `possible_switched_agent`

**8×8 coarse grid (inter-camera portals, stored in tracks table):**
- Purpose: Learn portal connections and timing statistics
- Derived: 4:1 downsampling from 32×32 (cell [15,20] → [3,5])
- **Benefits:**
  - Efficient adjacency matrix (64 cells vs 1024)
  - Robust to small variations (exit at fine [12,5] vs [13,5] = same coarse [3,1])
  - Portal timing statistics less noisy (aggregate across fine cells)

**Why both scales matter:**
- Fine grid: Human can see "person on right stayed on right" - system validates via fine grid consistency
- Coarse grid: Portal learning doesn't overfit to exact pixel positions
- LLM spatial reasoning: "Person on right side of frame" validated by fine grid (5,12) vs (2,12)

**Merge decision complexity analysis:**

**Time/space filtering (primary, O(1)):**
- Reject if time gap > 60s
- Reject if spatial distance > 2.5× expected (walking speed × time gap)
- Narrows candidates from 100s to 2-5 plausible matches
- **NOT sample-to-sample trajectory matching** (would be O(N²) where N=frames per track)
- Only compare track endpoints: start/end times, fine grid positions

**Re-ID disambiguation (O(1) per candidate):**
- Single 512-dim embedding dot product per candidate
- <1ms per comparison
- If best_similarity > 0.85 AND (best - second_best) > 0.15 → match
- If ambiguous → flag `possible_switched_agent`

**Concurrent matching (periodic, not continuous):**
- Every 30 seconds: compare all active + recently-ended tracks
- Typical: 10-20 active tracks across all cameras
- Cost: ~100ms every 30 seconds (negligible)
- Handles overlapping cameras, portals, occlusions, gaps with single algorithm

**Portal timing updates (at validation rate, not frame rate):**
- Only update from validated merges (~1/hour, not 10 FPS)
- Recompute rolling statistics (mean, P50, P90) from last 100 observations
- No EMA, no online learning per frame
- Adjacency matrix updated at human validation rate

**Solo track learning filter:**
```python
# Only learn portal timing from clean observations
learn_portal_timing = (
    track1.solo_track and           # No multi-person contamination
    track2.solo_track and
    not merge.possible_switched_agent and  # No identity ambiguity
    merge.validated                 # Human or high-confidence LLM approved
)
```

**Multi-person merge strategy (2-in-2-out):**
- Two agents on CameraA → two agents on CameraB
- Re-ID makes best 1:1 matches (even if similarity mediocre, e.g., 0.65)
- Flag both merges as `possible_switched_agent = true`
- **Prevents agent multiplication:** 2 agents in → 2 agents out (NOT 4 separate agents)
- Accept ambiguity with flag, exclude from learning, queue for human review
- False merge is recoverable (split and re-merge), agent multiplication is hard to untangle

**Computational guarantees:**
- Merge decision: O(1) regardless of track length (5 seconds or 5 minutes)
- No trajectory analysis, no frame-by-frame comparisons
- Portal stats: O(1) insert into rolling window, periodic recompute
- Concurrent matching: O(N) where N = active tracks (typically 10-20)

**Unbalanced group handling (Phase 3+):**

Scenarios where entry/exit counts don't match:
- 3 people enter portal, 1 exits (2 stayed behind or went elsewhere)
- 2 people enter, 0 exit (both stayed in intermediate space)
- 1 person enters, 2 exit (someone was already there)

**Schema support:**
- `tracks.group_id`: Links all tracks detected concurrently (same frame)
- `tracks.exit_type`: How track ended ('portal', 'stayed_behind', 'exit_boundary', etc.)
- `tracks.candidate_agents`: JSON array of possible agent IDs for ambiguous tracks
- `merges.group_merge_id`: Links all merges from same multi-person group event

**Detection workflow:**
```python
# At tracking time (multiple people in frame)
if detections_in_frame > 1:
    group_id = assign_group_id()
    for detection in detections:
        track = create_track(...)
        track.group_id = group_id
        track.candidate_agents = json.dumps(possible_agent_ids)  # [7, 8, 9]
        track.max_concurrent_agents = len(detections)

# At portal crossing
if track.exit_type == 'portal':
    entry_count = count_tracks(camera_a, group_id)
    exit_count = count_tracks(camera_b, group_id)
    if entry_count != exit_count:
        flag_unbalanced_group(group_id)
        queue_for_human_review()
```

**Review UI workflow:**
```python
# Human sees: "3 people entered FrontDoor, 1 exited to Driveway - resolve"
# Track 101 (exited): Which person was this?
#   Dropdown: Person A, Person B, Person C (from candidate_agents)
# Tracks 102, 103 (stayed behind): 
#   Mark as stayed_behind, don't merge

# After resolution:
track_101.agent_id = 7  # Person A
track_101.candidate_agents = None  # Resolved
track_102.exit_type = 'stayed_behind'
track_103.exit_type = 'stayed_behind'
```

**Queries enabled:**
```sql
-- Find unbalanced groups needing review
SELECT group_id, 
       camera,
       COUNT(*) as track_count,
       GROUP_CONCAT(exit_type) as exit_types
FROM tracks 
WHERE group_id IS NOT NULL
GROUP BY group_id, camera
HAVING COUNT(DISTINCT exit_type) > 1;  -- Mixed exit types = unbalanced

-- Get candidate agents for review dropdown
SELECT candidate_agents FROM tracks WHERE track_id = ?;
-- Returns: "[7,8,9]" → UI shows "Person A, Person B, Person C"
```

---

### Main Database: `metadata/marengo.db` (SQLite)

**Purpose:** Structured storage for tracks, agents, events, validations.

**Schema:**

```sql
-- Agents (persistent entities across cameras/time)
CREATE TABLE agents (
    agent_id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_class TEXT NOT NULL,             -- 'person', 'vehicle', 'animal'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen TIMESTAMP,
    identity_id INTEGER,                   -- Foreign key to identities table (face_library) if named
    status TEXT DEFAULT 'active',          -- 'active', 'archived', 'merged'
    merged_into_agent_id INTEGER           -- If merged, points to canonical agent
);
CREATE INDEX idx_agents_status ON agents(status);
CREATE INDEX idx_agents_identity ON agents(identity_id);

-- Track segments (observable windows on ONE camera)
-- A track belongs to ONE agent, but agent can have MANY tracks (across cameras/time)
CREATE TABLE tracks (
    track_id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id INTEGER NOT NULL,             -- Which agent this track belongs to
    camera TEXT NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    first_cell_row INTEGER,               -- 32×32 fine grid
    first_cell_col INTEGER,
    last_cell_row INTEGER,
    last_cell_col INTEGER,
    first_cell_8x8_row INTEGER,           -- 8×8 coarse grid (portal learning)
    first_cell_8x8_col INTEGER,
    last_cell_8x8_row INTEGER,
    last_cell_8x8_col INTEGER,
    detection_count INTEGER DEFAULT 0,
    max_confidence REAL,
    
    -- Multi-person tracking metadata
    solo_track BOOLEAN DEFAULT 1,         -- Only one agent in all frames (for clean learning)
    max_concurrent_agents INTEGER DEFAULT 1,  -- Max agents detected together in any frame
    
    -- ID-swap detection
    possibly_switched BOOLEAN DEFAULT 0,   -- ID swap flag (track may have wrong agent_id)
    
    -- Unbalanced group handling (Phase 3+)
    exit_type TEXT,                       -- 'portal', 'exit_boundary', 'occlusion', 'camera_edge', 'timeout', 'stayed_behind'
    exit_portal_id TEXT,                  -- Which portal if exit_type='portal'
    group_id INTEGER,                     -- Links tracks detected together (multi-person groups)
    candidate_agents TEXT,                -- JSON array: "[7,8,9]" for ambiguous assignment
    
    status TEXT DEFAULT 'active',          -- 'active', 'ended', 'merged'
    FOREIGN KEY (agent_id) REFERENCES agents(agent_id)
);
CREATE INDEX idx_tracks_camera ON tracks(camera);
CREATE INDEX idx_tracks_time ON tracks(start_time, end_time);
CREATE INDEX idx_tracks_agent ON tracks(agent_id);
CREATE INDEX idx_tracks_solo ON tracks(solo_track);  -- For learning filter queries
CREATE INDEX idx_tracks_group ON tracks(group_id);   -- For unbalanced group queries

-- Agent-Track relationship is 1:many
-- Get all tracks for agent: SELECT * FROM tracks WHERE agent_id = ?
-- Get agent for track: SELECT agent_id FROM tracks WHERE track_id = ?

-- ID-swap handling: If track has possibly_switched=1, it may point to wrong agent.
-- When swap detected:
--   1. Mark original track as possibly_switched=1
--   2. Create new track with new agent_id
--   3. Human review can correct agent_id assignments retroactively

-- Detection logs (sampled, not every frame)
CREATE TABLE detections (
    detection_id INTEGER PRIMARY KEY AUTOINCREMENT,
    track_id INTEGER NOT NULL,
    camera TEXT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    bbox_x1 INTEGER, bbox_y1 INTEGER, bbox_x2 INTEGER, bbox_y2 INTEGER,
    cell_row INTEGER, cell_col INTEGER,
    confidence REAL,
    class TEXT,
    frame_path TEXT,                       -- Path to JPEG
    is_keyframe BOOLEAN DEFAULT 0,         -- Keyframe sampling for long tracks
    FOREIGN KEY (track_id) REFERENCES tracks(track_id)
);
CREATE INDEX idx_detections_track ON detections(track_id);
CREATE INDEX idx_detections_time ON detections(timestamp);
CREATE INDEX idx_detections_keyframe ON detections(is_keyframe);  -- Fast keyframe-only queries

-- Keyframe sampling strategy (Phase 2+):
-- Problem: Long tracks (2 hours = 72,000 detections at 10 FPS) create millions of DB rows
-- Solution: Only log keyframes to database, keep all frames on disk
-- What gets logged (is_keyframe=1):
--   - First detection (track start)
--   - Last detection (track end)
--   - Every 100th frame (~10 seconds at 10 FPS)
--   - Direction changes (motion vector shift >45°)
--   - Quality peaks (best face crop in 10-second window)
-- Result: ~800 DB rows for 2-hour track instead of 72,000
-- Video reconstruction: Uses all 72,000 frames from disk (frame_path points to storage/)
-- Track summary: SELECT * FROM detections WHERE track_id=? AND is_keyframe=1  (fast, 800 rows)
-- Full history (rare): SELECT * FROM detections WHERE track_id=?  (slow, only if all logged)

-- Merge decisions
CREATE TABLE merges (
    merge_id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_track_id INTEGER NOT NULL,
    dest_track_id INTEGER NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    delta_t REAL,                          -- Time gap in seconds (negative if concurrent)
    portal_id TEXT,                        -- Portal used (if any)
    decision TEXT NOT NULL,                -- AUTO_MERGE, REVIEW_LLM, REVIEW_HUMAN, REJECT
    strong_evidence_count INTEGER,
    weak_evidence_count INTEGER,
    
    -- Similarity scores
    face_similarity REAL,                  -- Face match confidence (0.0-1.0)
    reid_similarity REAL,                  -- Re-ID match confidence (0.0-1.0)
    reid_second_best REAL,                 -- Second-best Re-ID candidate similarity
    reid_margin REAL,                      -- best - second_best (disambiguation confidence)
    
    -- Spatial metrics
    spatial_distance_m REAL,               -- Euclidean distance between track endpoints (meters)
    expected_distance_m REAL,              -- Based on walking_speed × delta_t
    plausibility_ratio REAL,               -- actual / expected (>2.5 = implausible)
    
    -- Multi-person metadata
    possible_switched_agent BOOLEAN DEFAULT 0,  -- Multi-person ambiguity flag
    source_concurrent_agents INTEGER,      -- Denormalized from tracks (for fast queries)
    dest_concurrent_agents INTEGER,
    group_merge_id INTEGER,                -- Links merges from same multi-person group
    
    -- Validation
    llm_confidence REAL,
    human_validated BOOLEAN DEFAULT 0,
    validation_outcome TEXT,               -- ACCEPT, REJECT, DEFER
    validation_timestamp TIMESTAMP,
    reviewed_by TEXT,                      -- 'human', 'llm', 'auto_face', 'auto_reid'
    correction_type TEXT,                  -- 'split', 'reassign', 'confirmed', NULL
    original_decision TEXT,                -- What system suggested before human override
    
    panel_path TEXT,                       -- Path to decision panels
    FOREIGN KEY (source_track_id) REFERENCES tracks(track_id),
    FOREIGN KEY (dest_track_id) REFERENCES tracks(track_id)
);
CREATE INDEX idx_merges_decision ON merges(decision);
CREATE INDEX idx_merges_validated ON merges(human_validated);
CREATE INDEX idx_merges_switched ON merges(possible_switched_agent);  -- For learning filter
CREATE INDEX idx_merges_group ON merges(group_merge_id);  -- For group resolution queries

-- Camera states (for debugging state transitions)
CREATE TABLE camera_states (
    state_id INTEGER PRIMARY KEY AUTOINCREMENT,
    camera TEXT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    old_state TEXT,
    new_state TEXT NOT NULL,               -- Standby, Armed, Active, Post
    trigger TEXT,                          -- What caused transition
    active_tracks INTEGER DEFAULT 0
);
CREATE INDEX idx_states_camera ON camera_states(camera);
CREATE INDEX idx_states_time ON camera_states(timestamp);

-- Events (multi-camera activity groups)
CREATE TABLE events (
    event_id INTEGER PRIMARY KEY AUTOINCREMENT,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    cameras TEXT,                          -- JSON array of camera names
    agent_ids TEXT,                        -- JSON array of agent_ids involved
    manifest_path TEXT,                    -- Path to event manifest JSON
    clip_path TEXT,                        -- Path to reconstructed MP4 (optional)
    status TEXT DEFAULT 'active'           -- 'active', 'closed', 'archived'
);
CREATE INDEX idx_events_time ON events(start_time);

-- Validations (ground truth for learning)
CREATE TABLE validations (
    validation_id INTEGER PRIMARY KEY AUTOINCREMENT,
    merge_id INTEGER NOT NULL,
    validated_by TEXT,                     -- 'llm' or 'user'
    outcome TEXT NOT NULL,                 -- ACCEPT, REJECT, DEFER
    confidence REAL,                       -- LLM confidence or 1.0 for human
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notes TEXT,
    FOREIGN KEY (merge_id) REFERENCES merges(merge_id)
);
CREATE INDEX idx_validations_merge ON validations(merge_id);
```

**Functions needed:**
```python
# Agents
def create_agent(agent_class: str) -> int  # Returns agent_id
def get_agent(agent_id: int) -> Agent
def merge_agents(source_agent_id: int, dest_agent_id: int)
def link_agent_to_identity(agent_id: int, identity_id: int)  # When user names them

# Tracks
def create_track(agent_id: int, camera: str, timestamp: datetime) -> int
def update_track(track_id: int, **kwargs)
def end_track(track_id: int, timestamp: datetime)
def get_track(track_id: int) -> Track
def get_agent_tracks(agent_id: int) -> List[Track]

# Detections
def log_detection(track_id: int, camera: str, timestamp: datetime, 
                  bbox: Tuple, cell: Tuple, confidence: float, class_name: str)
def get_track_detections(track_id: int) -> List[Detection]

# Merges
def record_merge_decision(source_track_id: int, dest_track_id: int,
                          evidence: dict, decision: str, **kwargs) -> int
def get_merge_decision(merge_id: int) -> MergeDecision
def update_merge_validation(merge_id: int, outcome: str, confidence: float, **kwargs)
def get_pending_reviews(limit: int = 100) -> List[MergeDecision]

# Camera states
def log_state_transition(camera: str, old_state: str, new_state: str, trigger: str)
def get_camera_current_state(camera: str) -> str
def get_state_history(camera: str, hours: int = 24) -> List[StateTransition]

# Events
def create_event(cameras: List[str], agent_ids: List[int]) -> int
def close_event(event_id: int, manifest_path: str, clip_path: str = None)
def get_active_events() -> List[Event]

# Validations
def record_validation(merge_id: int, validated_by: str, outcome: str, 
                      confidence: float, notes: str = None)
def get_validation_stats(days: int = 7) -> dict  # Accuracy, auto-merge rate
```

---

## Configuration Files

### 1. Stage Configuration

**File:** `config/stages/{STAGE}.yaml`

Example: `config/stages/DEV_SLOW.yaml`

```yaml
stage: DEV_SLOW
description: "Underpowered hardware testing (2 FPS, 5 cameras, no external AI)"

cameras:
  enabled: 5                      # Subset of cameras
  frame_rate: 2.0                 # FPS (will increase to 10 in Phase 2)
  profiles:
    sub: true
    main: false                   # No main profile in DEV_SLOW

inference:
  batch_size: 4
  max_concurrent_batches: 2
  yolo_model: "yolov8n.pt"        # Nano model
  
states:
  standby_ratio: 0.15
  armed_ratio: 0.6
  active_ratio: 1.0
  post_ratio: 0.4

external_ai:
  face_recognition: false
  lpr: false
  llm_adjudication: false

merging:
  enabled: false                  # No cross-camera merging in DEV_SLOW
  
retention:
  frames_days: 7
  metadata_days: 30
  
monitoring:
  web_dashboard: true
  port: 5000
```

**Functions needed:**
```python
def load_stage_config(stage: str) -> StageConfig
def validate_stage_config(config: StageConfig) -> List[str]  # Returns errors
def get_current_stage() -> str
def hot_reload_config()  # Reload without restart
```

---

### 2. Camera Definitions

**File:** `config/cameras.yaml`

```yaml
cameras:
  - name: FrontDoor
    bi_shortname: d5442ip41
    enabled: true
    resolution: [1920, 1080]
    ignore_zones:
      - name: tree_sway
        polygon: [[0.1, 0.2], [0.15, 0.2], [0.15, 0.3], [0.1, 0.3]]
    
    # Camera calibration metadata (Phase 4+, optional)
    last_calibration_check: "2025-11-08T10:00:00Z"
    reference_frame_hash: "a3f5b8c2..."  # Perceptual hash for movement detection
    clock_offset_ms: 0                   # If BI timestamps unreliable (usually 0)
    
  - name: Driveway
    bi_shortname: d5442ip42
    enabled: true
    resolution: [1920, 1080]
```

**Editing:** Via `config_camera_params.py` UI tool (not manual YAML edit).

**Note:** Grid is fixed 32×32 for all cameras. BlueIris handles camera connectivity/IPs. BlueIris also provides timestamps, so clock_offset_ms typically unused (BI normalizes camera clocks).

---

### 3. Portal Configuration

**File:** `config/portals.yaml`

```yaml
portals:
  - id: FrontDoor_to_Driveway
    camera_a: FrontDoor
    camera_b: Driveway
    cell_a: [0, 15]               # 32×32 cell coordinates
    cell_b: [0, 16]
    direction: bidirectional      # or 'a_to_b', 'b_to_a'
    typical_time: 2.0             # seconds (user-configured initial)
    max_time: 10.0
    exit: false
    enabled: true
    notes: "Front door to driveway path"
    
  - id: Garage_to_Outside
    camera_a: Garage
    camera_b: Outside
    cell_a: [5, 10]
    cell_b: [5, 11]
    direction: bidirectional
    typical_time: 3.0
    max_time: 15.0
    exit: true                    # Exit portal (track termination)
    enabled: true
    notes: "Garage exit to property boundary"
```

**Editing:** Via `config_portal_editor.py` UI tool with visual overlay.

**Functions needed:**
```python
def load_portal_config() -> PortalConfig
def save_portal_config(config: PortalConfig)
def add_portal(camera_a: str, camera_b: str, cell_a: Tuple, cell_b: Tuple, **kwargs) -> str
def remove_portal(portal_id: str)
def enable_portal(portal_id: str, enabled: bool)
def update_portal(portal_id: str, **kwargs)
def get_portals_for_camera(camera: str) -> List[Portal]
def find_portal_by_cells(camera_a: str, cell_a: Tuple, 
                         camera_b: str, cell_b: Tuple) -> Optional[str]
```

---

### 4. Thresholds Configuration

**File:** `config/thresholds.yaml`

```yaml
detection:
  person_confidence: 0.55
  vehicle_confidence: 0.60
  animal_confidence: 0.50
  
tracking:
  iou_threshold: 0.30
  max_cell_jump: 3                # Max cells per frame (anti-teleport)
  sustained_duration: 3.0         # Seconds for Active transition
  
merging:
  auto_merge:
    strong_evidence_min: 2
    face_similarity_min: 0.75     # Cosine similarity
  review_llm:
    strong_evidence_min: 1
    weak_evidence_min: 1
  review_human:
    max_candidates: 3             # >3 → human review
    
face_recognition:
  match_threshold: 0.60           # Auto-merge if similarity >= this
  reject_threshold: 0.80          # Reject if similarity <= this
  quality_min: 0.55
  
llm:
  accept_confidence: 0.80         # Accept YES if confidence >= this
  reject_confidence: 0.70         # Reject NO if confidence >= this
  
timing:
  portal_discovery_window: 5.0    # Seconds for suggesting new portals
  wide_discovery_window: 60.0     # Fallback if no portals configured
  gap_tolerance: 0.3              # Max gap in track continuity
```

**Editing:** Via `config_threshold_tuner.py` UI with real-time preview.

---

### 5. External Services

**File:** `config/services.yaml`

```yaml
codeproject_ai:
  instances:
    - name: GPU_Primary
      endpoint: "http://192.168.60.10:32168"
      priority: 1
      enabled: true
      models:
        face_recognition: true
        lpr: true
        scene_classification: false
      health:
        timeout_sec: 5.0
        retry_count: 3
        circuit_breaker_threshold: 5
        
    - name: CPU_Fallback
      endpoint: "http://192.168.60.11:32168"
      priority: 2
      enabled: true
      models:
        face_recognition: true
        lpr: false

llm:
  provider: gemini
  model: gemini-flash-2.0
  api_key_secret: "GEMINI_API_KEY"    # Reference to encrypted secret
  temperature: 0.0
  max_tokens: 500
  timeout_sec: 10.0
```

---

## File Formats & Serialization

### JSON for Human-Readable Config
- Camera definitions
- Portal configuration
- Grid parameters
- Event manifests

### YAML for Stage/Threshold Config
- More human-friendly than JSON
- Comments allowed
- Hot-reloadable

### SQLite for Structured Metadata
- Tracks, agents, merges, validations
- Queryable, indexed, ACID guarantees
- Backup-friendly (single file)

### JSONL (JSON Lines) for Append-Only Logs
- Detection logs (high volume)
- Track logs
- Merge decision logs
- Easy to append, easy to parse line-by-line

### NumPy `.npy` for Embeddings (Optional)
- Face embeddings (512-dim float32)
- Re-ID fingerprints
- Faster than JSON for large arrays
- Alternative: Store as BLOB in SQLite

### MessagePack for Wire Protocol (Optional Phase 2+)
- Faster than JSON for inter-process communication
- Compressed, binary format
- Good for high-frequency frame metadata

---

## Function Inventory by Phase

### Phase 0: Foundation
```python
# Frame acquisition
fetch_jpeg_from_bi(camera: str, profile: str) -> bytes
save_frame(camera: str, timestamp: datetime, profile: str, jpeg_bytes: bytes) -> str
prune_old_frames(older_than_days: int)

# Detection
run_yolo_batch(frames: List[np.ndarray]) -> List[List[Detection]]
filter_by_confidence(detections: List[Detection], threshold: float) -> List[Detection]

# Storage
initialize_storage_structure()
create_daily_frame_dirs(camera: str, date: str)

# Config
load_stage_config(stage: str) -> StageConfig
load_camera_config() -> CameraConfig
hot_reload_config()

# Monitoring
log_system_metric(metric: str, value: float)
get_camera_stats(camera: str) -> dict
```

### Phase 1: Single-Camera Tracking
```python
# Grid
load_camera_grid_params(camera: str) -> GridParams
compute_grid_cell(bbox: Tuple, grid: GridParams) -> Tuple[int, int]
is_point_in_ignore_zone(camera: str, x: float, y: float) -> bool

# Height calibration
load_height_grid(camera: str) -> HeightGrid
update_cell_height(grid: HeightGrid, row: int, col: int, height: float)
get_expected_height(grid: HeightGrid, row: int, col: int) -> Optional[float]

# Tracking
associate_detections_iou(prev_tracks: List[Track], 
                         new_detections: List[Detection]) -> List[Tuple]
detect_id_swap(track: Track, new_detection: Detection) -> bool
create_track(agent_id: int, camera: str, timestamp: datetime) -> int
update_track(track_id: int, detection: Detection)
end_track(track_id: int, timestamp: datetime, exit_type: str = None)

# Multi-person group tracking
assign_group_id() -> int
    # Generate unique group_id for tracks detected together
detect_multi_person_frame(detections: List[Detection]) -> bool
    # Returns True if multiple people in same frame
set_candidate_agents(track_id: int, agent_ids: List[int])
    # Store JSON array of possible agent IDs for ambiguous tracks
get_candidate_agents(track_id: int) -> List[int]
    # Parse candidate_agents JSON for UI dropdown

# Keyframe sampling (long tracks)
should_log_detection(track: Track, detection: Detection, frame_number: int) -> bool
    # Determines if detection should be logged to DB (keyframe) or skipped
    # Always log: first, last, every 100th, direction changes, quality peaks
mark_detection_keyframe(detection_id: int)
    # Set is_keyframe=1 for important detections

# State machine
transition_camera_state(camera: str, new_state: str, trigger: str)
should_transition_to_armed(camera: str, detections: List[Detection]) -> bool
should_transition_to_active(camera: str, track: Track) -> bool

# Database
create_agent(agent_class: str) -> int
log_detection(track_id: int, camera: str, timestamp: datetime, bbox: Tuple, 
              is_keyframe: bool = False, ...)
get_active_tracks(camera: str) -> List[Track]
```

### Phase 2: Cross-Camera Merging
```python
# Identity management (CRITICAL: prevents duplicate people/vehicles)
get_identity_suggestions(entity_type: str = "person", limit: int = 10) -> List[dict]
    # Returns recent/common identities for dropdown autocomplete
    # Sorted by last_seen DESC, total_tracks DESC
create_or_get_identity(name: str, entity_type: str) -> int
    # Case-insensitive, normalized lookup (prevents "John Doe" vs "john doe")
    # Returns existing identity_id or creates new one
search_identities(query: str, entity_type: str, limit: int = 20) -> List[dict]
    # For autocomplete search box
record_merge_with_identity(merge_id: int, identity_name: str, 
                           source_track: Track, dest_track: Track)
    # Links validated merge to identity, adds face embeddings to library
rename_identity(identity_id: int, new_name: str)
    # Admin tool for corrections
merge_identities(keep_id: int, duplicate_id: int)
    # Admin cleanup: merge duplicate identity records if they slip through
    
# Portal configuration
load_portal_config() -> PortalConfig
get_portals_for_camera(camera: str) -> List[Portal]
find_portal_by_cells(camera_a: str, cell_a: Tuple, 
                     camera_b: str, cell_b: Tuple) -> Optional[str]

# Portal timing
load_portal_stats() -> PortalStats
get_portal_window(portal_id: str) -> Tuple[float, float]
record_portal_observation(portal_id: str, delta_t: float, validated: bool)

# Merge candidate finding
find_merge_candidates_via_portals(track: Track) -> List[Track]
find_merge_candidates_wide_discovery(track: Track, max_delta_t: float) -> List[Track]

# Evidence scoring
count_evidence(source_track: Track, dest_track: Track) -> Tuple[int, int]  # (strong, weak)
check_portal_match(source_track: Track, dest_track: Track) -> bool
check_timing_plausible(delta_t: float, portal_id: str) -> bool

# Face recognition
extract_face_crop(frame: np.ndarray, bbox: Tuple) -> Optional[np.ndarray]
compute_face_embedding(crop: np.ndarray) -> np.ndarray
match_face_to_library(embedding: np.ndarray, threshold: float) -> Optional[Tuple[int, float]]
    # Returns (identity_id, similarity) if match found
add_face_to_library(identity_id: int, embedding: np.ndarray, quality: float, **kwargs)
    # NOTE: Uses identity_id, not person_id

# Panel generation
create_merge_panel(source_track: Track, dest_track: Track) -> int
generate_panel_images(track: Track, count: int = 6) -> List[np.ndarray]
save_panel_composite(merge_id: int, panel_a: np.ndarray, panel_b: np.ndarray)

# LLM integration
call_llm_for_merge(merge_id: int, panel_a_path: str, panel_b_path: str, 
                   evidence: dict) -> dict
parse_llm_response(response: dict) -> Tuple[str, float, str]  # (answer, confidence, reasons)

# Decision recording
record_merge_decision(source_track_id: int, dest_track_id: int, 
                      evidence: dict, decision: str, **kwargs) -> int
get_pending_reviews(decision_type: str = None) -> List[MergeDecision]

# Validation
record_human_validation(merge_id: int, outcome: str, notes: str = None)
record_llm_validation(merge_id: int, outcome: str, confidence: float, reasons: str)
update_portal_stats_from_validation(merge_id: int)  # Learn from ground truth

# Unbalanced group resolution (Phase 3+)
flag_unbalanced_group(group_id: int)
    # Mark group as needing human review (entry count != exit count)
get_unbalanced_groups() -> List[dict]
    # Query groups with mismatched entry/exit counts
resolve_track_identity(track_id: int, chosen_agent_id: int)
    # Human selects which agent from candidate_agents list
mark_track_stayed_behind(track_id: int)
    # Mark track as exit_type='stayed_behind' (didn't continue through portal)
get_group_tracks(group_id: int) -> List[Track]
    # Get all tracks from same multi-person group
```

### Phase 3: Uncertainty & Vehicles
```python
# Multi-state tracking
set_agent_possible_locations(agent_id: int, locations: List[dict])
get_agent_possible_locations(agent_id: int) -> List[dict]
collapse_uncertainty(agent_id: int, observed_location: dict)

# Vehicle containers
associate_person_with_vehicle(person_track: Track, vehicle_track: Track, 
                               evidence: dict) -> int
rule_out_vehicle_association(person_agent_id: int, vehicle_agent_id: int, 
                              reason: str)
get_vehicle_occupancy_candidates(vehicle_track: Track) -> List[Track]

# Exit portals
is_exit_portal(portal_id: str) -> bool
terminate_track_at_exit(track_id: int, portal_id: str)
```

### Phase 4: Maturity & Re-ID (Optional)
```python
# Portal window tightening
tighten_portal_windows(min_observations: int = 50)
should_use_learned_window(portal_id: str, min_confidence: float = 0.7) -> bool

# Re-ID (optional, deferred)
# NOTE: Re-ID fingerprints will also reference identities table (same as face_embeddings)
# Schema: reid_fingerprints(fingerprint_id, identity_id, camera_id, embedding, lighting, quality, ...)
extract_reid_embedding(frame: np.ndarray, bbox: Tuple) -> np.ndarray
add_reid_fingerprint(identity_id: int, camera: str, embedding: np.ndarray, **kwargs)
match_reid(embedding: np.ndarray, camera: str, threshold: float) -> Optional[Tuple[int, float]]
prune_reid_fingerprints(max_per_agent_per_camera: int = 5)
```

---

## Library Dependencies

### Core Libraries
```python
# Already decided/standard
opencv-python >= 4.8.0        # Frame manipulation, grid overlay
numpy >= 1.24.0               # Arrays, embeddings
ultralytics >= 8.0.0          # YOLOv8
torch >= 2.0.0                # YOLO backend
pillow >= 10.0.0              # Image I/O

# Database & serialization
sqlite3                       # Built-in, metadata storage
pyyaml >= 6.0                 # Config files
msgpack >= 1.0.0              # Optional wire protocol (Phase 2+)

# Web frameworks
flask >= 2.3.0                # Web UIs (dashboards, review queue)
flask-cors >= 4.0.0           # CORS for web UIs

# External AI
requests >= 2.31.0            # HTTP calls to CodeProject.AI, LLM APIs
google-generativeai >= 0.3.0  # Gemini API client

# GUI tools
tkinter                       # Built-in, for config tools
matplotlib >= 3.7.0           # Plotting (optional, for calibration charts)

# Utilities
python-dateutil >= 2.8.0      # Timestamp parsing
tqdm >= 4.65.0                # Progress bars
click >= 8.1.0                # CLI framework for tools

# Crypto (secrets)
cryptography >= 41.0.0        # Encrypt API keys
```

### Optional (Phase 4+)
```python
# Re-ID models
faiss-cpu >= 1.7.0            # Fast embedding search
scikit-learn >= 1.3.0         # Clustering for identity merging
```

---

## Configuration UI Tools

### 1. Portal Editor (`tools/config_portal_editor.py`)

**Purpose:** Visual UI to define portals between cameras.

**Features:**
- Side-by-side camera views with 32×32 grid overlay
- Click to select portal cells on each camera
- Set typical_time, max_time, direction, exit flag
- Preview portal candidates (suggested from observations)
- Save to `config/portals.yaml`

**UI Flow:**
1. Select camera A → show live feed with grid
2. Click cell for portal exit point
3. Select camera B → show live feed with grid
4. Click cell for portal entry point
5. Set timing parameters (typical, max)
6. Set direction (bidirectional, a_to_b, b_to_a)
7. Mark as exit portal if boundary crossing
8. Save → validates and writes to YAML

**Functions:**
```python
def show_camera_with_grid(camera: str) -> np.ndarray
def draw_grid_overlay(frame: np.ndarray, grid_params: GridParams) -> np.ndarray
def highlight_cell(frame: np.ndarray, row: int, col: int, color: Tuple)
def get_click_cell(x: int, y: int, grid_params: GridParams) -> Tuple[int, int]
def validate_portal(portal: Portal) -> List[str]  # Returns errors
def save_portal_to_yaml(portal: Portal)
```

---

### 2. Camera Grid Configurator (`tools/config_camera_params.py`)

**Purpose:** View camera feed and configure ignore zones.

**Features:**
- Live camera feed with fixed 32×32 grid overlay
- Draw polygon ignore zones (click points, right-click to close)
- Save ignore zones to `state/camera_params/{camera}_params.json`

**UI Flow:**
1. Select camera → show live feed with 32×32 grid overlay
2. Press 'g' to toggle grid on/off
3. Press 'i' to enter ignore zone mode
4. Click polygon points to define zone
5. Right-click to close polygon
6. Name zone and set reason
7. Save → writes to camera params JSON

**Functions:**
```python
def load_camera_feed(camera: str) -> cv2.VideoCapture
def draw_uniform_grid(frame: np.ndarray, grid_size: Tuple[int, int]) -> np.ndarray
def draw_ignore_zones(frame: np.ndarray, zones: List[IgnoreZone]) -> np.ndarray
def add_ignore_zone_interactive(camera: str) -> IgnoreZone
def save_camera_params(camera: str, params: CameraParams)
```

**Note:** Grid is fixed 32×32 uniform - no adjustable perspective parameters. Simpler, more stable across camera changes.

---

### 3. Threshold Tuner (`tools/config_threshold_tuner.py`)

**Purpose:** Adjust confidence/timing thresholds with real-time feedback.

**Features:**
- Slider UI for all thresholds (detection, tracking, merging, face, LLM)
- Live preview: apply thresholds to recent logged data
- Show auto-merge rate, review queue size, false positive rate
- Save to `config/thresholds.yaml`

**Functions:**
```python
def load_recent_merge_decisions(hours: int = 24) -> List[MergeDecision]
def simulate_threshold_change(decisions: List[MergeDecision], 
                               new_thresholds: dict) -> dict  # Returns stats
def plot_threshold_impact(stats: dict)  # Auto-merge rate vs threshold
def save_thresholds_to_yaml(thresholds: dict)
```

---

### 4. Config Validator (`tools/config_validator.py`)

**Purpose:** Validate all YAML configs, check for errors before hot-reload.

**Functions:**
```python
def validate_all_configs() -> dict  # Returns errors per file
def validate_cameras_yaml(path: str) -> List[str]
def validate_portals_yaml(path: str) -> List[str]
def validate_stage_yaml(path: str) -> List[str]
def check_portal_references(portals: List[Portal], cameras: List[str]) -> List[str]
def hot_reload_if_valid() -> bool  # Returns True if reload successful
```

---

### 5. State Backup Tool (`tools/config_backup.py`)

**Purpose:** Backup/restore learned state (height grids, portal stats, face library).

**Functions:**
```python
def create_backup(backup_name: str = None) -> str  # Returns backup path
def list_backups() -> List[str]
def restore_backup(backup_path: str) -> bool
def export_to_json(backup_path: str) -> dict  # For portability
def import_from_json(json_path: str) -> bool
```

---

## Data Migration & Versioning

### Version Field in All JSON/YAML
```json
{
  "version": "1.0",
  "schema": "height_grid_v1",
  // ... data
}
```

### Migration Functions
```python
def migrate_height_grid_v1_to_v2(old_data: dict) -> dict
def migrate_portal_stats_v1_to_v2(old_data: dict) -> dict
def detect_schema_version(data: dict) -> str
def auto_migrate_if_needed(data: dict, expected_version: str) -> dict
```

### Database Schema Migrations
```sql
-- Use Alembic or simple versioning table
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO schema_version (version) VALUES (1);
```

**Migration scripts:** `migrations/001_initial.sql`, `migrations/002_add_reid.sql`, etc.

---

## Rollout Notes

### Phase 0-1: Start at 2 FPS (DEV_SLOW)
- Prove acquisition and detection work
- Low resource usage for initial development
- Frame naming supports arbitrary FPS (millisecond precision)

### Phase 2: Increase to 10 FPS (DEV_FULL, PROD)
- After basic tracking proven
- Face recognition needs higher temporal resolution
- Portal crossing detection benefits from denser sampling
- Retention policy keeps more frames

**Config change:**
```yaml
# DEV_SLOW.yaml
frame_rate: 2.0

# DEV_FULL.yaml  
frame_rate: 10.0  # Phase 2 increase
```

**Functions already support variable FPS:**
- `save_frame()` uses millisecond-precision timestamps
- Detection logging samples at configured rate
- No hardcoded 10 FPS assumptions

---

## Data Types Summary

### Learned Data (State)
| Type | Format | Retention | Backup |
|------|--------|-----------|--------|
| Height grids (32×32) | JSON | Indefinite | Weekly |
| Portal timing stats | JSON | Indefinite | Weekly |
| Face embeddings | SQLite + BLOB | Indefinite | Weekly |
| Camera grid params | JSON | Indefinite | Weekly |
| Re-ID fingerprints | SQLite + BLOB | Indefinite | Weekly |

### Raw Data (Media)
| Type | Format | Retention | Backup |
|------|--------|-----------|--------|
| JPEG frames (sub) | JPEG | 7-30 days | None (reconstructible) |
| JPEG frames (main) | JPEG | 7-30 days | None |
| Face crops | JPEG | 90 days | None (in library if kept) |
| Merge panels | JPEG | Indefinite | None (audit trail) |
| MP4 clips | MP4 | 30 days | Optional |

### Metadata (Logs)
| Type | Format | Retention | Backup |
|------|--------|-----------|--------|
| Tracks | SQLite | 90 days | Daily |
| Detections (sampled) | SQLite | 30 days | None |
| Merge decisions | SQLite | Indefinite | Daily |
| Validations | SQLite | Indefinite | Daily |
| Events | SQLite | 90 days | Daily |
| Camera states | SQLite | 30 days | None |

### Configuration (User)
| Type | Format | Retention | Backup |
|------|--------|-----------|--------|
| Stage configs | YAML | Indefinite | Git |
| Camera definitions | YAML | Indefinite | Git |
| Portal config | YAML | Indefinite | Git |
| Thresholds | YAML | Indefinite | Git |
| Services | YAML | Indefinite | Git |
| Secrets | Encrypted | Indefinite | Manual |

---

## Pre-Positioned Data Fields

### Detection Log (Even if Not Used Until Phase 2)
```python
{
    "detection_id": 12345,
    "track_id": 42,
    "timestamp": "2025-11-07T14:30:00.123Z",
    "bbox": [100, 200, 300, 400],
    "cell_32x32": [15, 20],
    "cell_8x8": [3, 5],           # Pre-positioned for Phase 2
    "confidence": 0.87,
    "class": "person",
    "frame_path": "storage/frames/...",
    
    # Face-related (Phase 2)
    "face_detected": false,
    "face_quality": null,
    "face_embedding_id": null,
    
    # Re-ID (Phase 4, deferred)
    "reid_embedding_id": null
}
```

### Track Record (Pre-Position for Later Phases)
```python
{
    "track_id": 42,
    "agent_id": 7,
    "camera": "FrontDoor",
    "start_time": "2025-11-07T14:30:00Z",
    "end_time": "2025-11-07T14:30:45Z",
    "first_cell": [0, 5],
    "last_cell": [31, 28],
    "detection_count": 45,
    "max_confidence": 0.92,
    
    # ID-swap tracking (Phase 1)
    "possibly_switched": false,
    "switch_suspected_at": null,
    
    # Multi-state (Phase 3)
    "possible_locations": [],      # Empty until Phase 3
    
    # Vehicle association (Phase 3)
    "associated_vehicle_track_id": null,
    "association_confidence": null
}
```

### Agent Record (Pre-Position for Identification)
```python
{
    "agent_id": 7,
    "agent_class": "person",
    "created_at": "2025-11-07T14:30:00Z",
    "last_seen": "2025-11-07T14:30:45Z",
    
    # Face identification (Phase 2)
    "person_id": null,             # Links to face_library
    "identified": false,
    
    # Vehicle linkage (Phase 3)
    "vehicle_agent_id": null,
    
    # Re-ID (Phase 4)
    "reid_fingerprint_count": 0,
    
    # Status
    "status": "active",            # 'active', 'archived', 'merged'
    "merged_into_agent_id": null
}
```

---

## Critical Bootstrap Requirements

### Phase 0 Must Log:
- All detections (sampled) with bbox, cell, confidence
- Track start/end times with first/last cells
- Camera state transitions
- Frame paths (for reconstruction)

### Phase 1 Must Add:
- Height observations per cell (for calibration)
- ID-swap flags (for debugging)
- Track continuity gaps

### Phase 2 Must Add:
- Face crop paths and quality scores
- Portal usage (which portals actually used)
- Merge decision evidence breakdown
- LLM responses (for calibration)

### Phase 3 Must Add:
- Possible locations (uncertainty tracking)
- Vehicle associations
- Exit portal usage

### Phase 4 May Add (Optional):
- Re-ID embeddings
- Portal window tightening stats

---

## Summary

**All data structures are now pre-planned.** We won't realize in Phase 3 that we should have been logging face quality scores from Phase 0. Everything is positioned early with nullable/empty fields that get populated as features roll out.

**Configuration via UI tools, not manual YAML editing.** Portal editor, grid configurator, threshold tuner all provide validation and prevent user errors.

**Storage is organized by purpose:** Learned data (state), raw data (frames), metadata (logs), configuration (user-editable). Each has appropriate retention, backup, and access patterns.

**Functions are inventoried by phase** so we know exactly what to implement when. No surprises.

**Libraries are locked in** — no mid-project dependency changes.

**FPS increase planned for Phase 2** — frame storage and detection logging already support arbitrary FPS via millisecond-precision timestamps.

Ready to build.
