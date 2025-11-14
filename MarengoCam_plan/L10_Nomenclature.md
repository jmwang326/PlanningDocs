# MarengoCam Nomenclature Guide

**Purpose:** Define standard terminology used throughout the project to ensure consistency across code, documentation, and communication.

**Last Updated:** 2025-11-14

---

## Core Architecture

### LocalTrack
**What it is:** One observation from YOLO tracking in one 12-second chunk on one camera. **This is the fundamental unit of the system.**

**Duration:** Variable (up to 12 seconds). Track ends when agent leaves frame or chunk ends.

**Contains:**
- `local_id` - YOLO tracking ID (chunk-scoped)
- `camera_id` - Which camera
- `chunk_id` - Which 12s chunk
- `detections[]` - Frame-by-frame bounding boxes
- `start_time`, `end_time` - UTC timestamps
- `entry_cell`, `exit_cell` - 6x6 grid cells

**Relationships:**
- `previous_track` - Same camera, previous chunk (None if first appearance)
- `next_track` - Same camera, next chunk (None if agent left)
- `merged_tracks[]` - Other cameras, same agent
- `concurrent_tracks[]` - Same camera, different agents

**Identity:**
- `global_agent_id` - Which agent (if known)
- `identity_candidates` - Set of possible agents (if uncertain)

**Context flags:**
- `inside_vehicle` - Detected through vehicle window
- `at_portal` - Started/ended at portal
- `property_entry` - Came from outside property
- `still_visible_at_end` - Other agents present when ended
- `exit_reason` - "left_frame" | "chunk_boundary" | "portal_crossed"

**Usage Examples:**
```python
track = LocalTrack(
    local_id=3,
    camera_id="FrontDoor",
    chunk_id=42,
    start_time=1699363200.0,
    end_time=1699363207.5,  # Only 7.5s - agent left mid-chunk
    exit_reason="left_frame",
    next_track=None,  # No continuation
    global_agent_id=123
)
```

**Always write as:** "LocalTrack" (code), "track" (informal)

**NOT called:** "track segment", "observation", "clip" (use "LocalTrack" consistently)

---

### GlobalAgent
**What it is:** A query result representing one entity (person, vehicle, animal) across all cameras and time. **Not a stored object** - computed from LocalTracks.

**How to get agent info:**
```python
# Get all tracks for agent
timeline = db.query(LocalTrack).filter(global_agent_id == 123).order_by(start_time).all()

# Get current state
latest_track = db.query(LocalTrack).filter(global_agent_id == 123).order_by(start_time.desc()).first()
if latest_track.inside_vehicle:
    state = "in_vehicle"
elif latest_track.at_portal:
    state = f"in_structure({latest_track.at_portal})"
else:
    state = f"visible({latest_track.camera_id})"
```

**Properties (all derived from LocalTracks):**
- Timeline - all LocalTracks with this global_agent_id
- Current state - computed from latest LocalTrack
- Last seen - latest LocalTrack.end_time
- Face library - managed separately

**Always write as:** "GlobalAgent" (code), "agent" (informal)

**NOT:** "Agent object", "Agent entity" (emphasize it's computed, not stored)

---

### Detection
**What it is:** Single YOLO inference result for one frame, one bounding box.

**Contains:**
- `timestamp` - UTC
- `bbox` - (x, y, width, height) in pixels
- `class` - "person" | "vehicle" | "animal"
- `confidence` - YOLO detection confidence [0.0, 1.0]
- `cell_6x6` - Grid cell containing bbox centroid

**Relationship:** Multiple detections form a LocalTrack.

**Usage Examples:**
```python
detection.class = "person"  # Lowercase
detection.confidence = 0.87
detection.cell_6x6 = (3, 4)  # Row, col
```

**Always write as:** "detection" (lowercase singular), "detections" (plural)

**NOT called:** "inference", "result", "bbox" (use "detection" as noun)

---

## Handlers (Processing Components)

### ChunkProcessor
**What it is:** Converts 12s video chunks into LocalTracks.

**Input:** Video chunk (12s, one camera)
**Output:** List of LocalTracks
**Timing:** Every 10s (overlapping chunks)

**Always write as:** "ChunkProcessor" (capitalized)

---

### TemporalLinker
**What it is:** Links LocalTracks across chunk boundaries (same camera).

**Input:** Current chunk LocalTracks + previous chunk LocalTracks
**Output:** Updated previous_track/next_track pointers
**Timing:** After ChunkProcessor

**Always write as:** "TemporalLinker" (capitalized)

---

### IdentityResolver
**What it is:** Assigns global_agent_id to LocalTracks (cross-camera merging).

**Input:** LocalTrack without global_agent_id
**Output:** global_agent_id OR identity_candidates
**Timing:** Periodic (batched)

**Always write as:** "IdentityResolver" (capitalized)

---

### EvidenceProcessor
**What it is:** Resolves uncertain LocalTracks when evidence arrives.

**Input:** Evidence + LocalTracks with identity_candidates
**Output:** Updated global_agent_id
**Timing:** Async (event-driven)

**Always write as:** "EvidenceProcessor" (capitalized)

---

## Grid Systems

### 6x6 Inter-Camera Grid
**Purpose:** Learn and store minimum travel time between camera pairs to enable plausible track merging.

**Specifications:**
- **Matrix:** 6x6 maintained for each directed camera pair and agent class
- **Cell Value:** Minimum observed travel time (milliseconds)
- **Overlap:** Time of `0` indicates direct camera overlap zone
- **Unlearned:** Value of `-1` represents unlearned path

**Learning:**
- Populated from validated merges (clean data only)
- Always stores lowest observed time
- Bidirectional (separate for each direction)

**Usage Examples:**
```python
travel_time = grid.get_time('FrontDoor', 'Driveway', 'person', (5,2), (0,3))
if travel_time == 0:
    # Overlap zone
elif travel_time > 0:
    # Learned transition
else:
    # Unlearned (-1)
```

**Always write as:** "6x6 grid" or "inter-camera grid"

**NOT:** "adjacency matrix", "spatial grid"

---

### Entry Cell
**Purpose:** 6x6 grid cell where LocalTrack first appears.

**Logic:** `entry_cell = grid_position(detections[0].bbox_center)`

**Usage:** Destination point for travel time computation.

---

### Exit Cell
**Purpose:** 6x6 grid cell where LocalTrack last appears.

**Logic:** `exit_cell = grid_position(detections[-1].bbox_center)`

**Usage:** Source point for travel time computation.

---

## Camera States

### Standby
**Definition:** Quiet baseline, no recent activity.

**Behavior:**
- Acquisition: 10 FPS (sub profile only)
- Inference: Sparse (motion gate + minimal YOLO)
- Frame persistence: No (25s buffer only)

**Transitions:**
- → Armed: Motion detected

**Always write as:** "Standby" (capitalized)

---

### Armed
**Definition:** Recent activity detected, agent not yet confirmed.

**Behavior:**
- Acquisition: 10 FPS (sub profile)
- Inference: Moderate (confirming sustained movement)
- Frame persistence: Yes (all sub frames)

**Transitions:**
- → Active: Sustained movement ≥3s
- → Standby: Motion stops before threshold

**Always write as:** "Armed" (capitalized)

---

### Active
**Definition:** At least one persistent agent track confirmed.

**Behavior:**
- Acquisition: 10 FPS (sub + main profiles)
- Inference: Full
- Frame persistence: Yes (sub + main)

**Transitions:**
- → Post: All tracks lost for quiet window

**Always write as:** "Active" (capitalized)

---

### Post
**Definition:** Cool-down after agent leaves frame.

**Behavior:**
- Acquisition: 10 FPS (sub profile)
- Inference: Moderate
- Frame persistence: Yes (sub frames)
- Duration: post_roll_s (10-15s TBD)

**Transitions:**
- → Active: New detection
- → Standby: Post-roll expires

**Always write as:** "Post" (capitalized)

---

## Identity Resolution

### Auto-Merge
**Definition:** Automatic merge without human/LLM review.

**Triggers:**
- High-confidence face match (≥0.75)
- Grid overlap (<0.4s) + single agent
- Grid transition + alibi passes + single agent

**Outcome:** LocalTrack.global_agent_id assigned immediately.

**Always write as:** "Auto-Merge" (hyphenated, capitalized)

---

### Queue for Review
**Definition:** Plausible merge candidate, needs human or LLM adjudication.

**Triggers:**
- Time/space plausible but ambiguous
- Multiple candidates (identity_candidates set)
- No strong evidence

**Outcome:** LocalTrack.identity_candidates populated, waits for evidence.

**Always write as:** "Queue for Review" or "queued"

---

### Reject
**Definition:** Not the same agent (definitive).

**Evidence:**
- Face mismatch
- Spatial/temporal impossibility
- Alibi check fails (seen elsewhere)

**Outcome:** LocalTracks remain with different global_agent_id values.

**Always write as:** "Reject" (capitalized)

---

## External AI Services

### CodeProject.AI
**What it is:** External AI inference service for specialized models.

**Services:**
- `face_recognition` - Trainable face library
- `lpr` - License Plate Recognition

**Always write as:** "CodeProject.AI"

**NOT:** "CPAI", "CP.AI"

---

### Face Recognition
**Method:** CodeProject.AI trainable library.

**Workflow:**
1. Agent confirmed via merge
2. Best face crops extracted
3. Registered with name
4. Future detections auto-recognized (≥0.75 → auto-merge)

**Always write as:** "face recognition" (lowercase)

**NOT:** "facial recognition", "FR"

---

### LPR (License Plate Recognition)
**Method:** CodeProject.AI model.

**Priority:** Higher than face for vehicles.

**Always write as:** "LPR" (uppercase acronym)

---

## Frame Terminology

### Sub Profile
**Definition:** Low-res JPEG stream from Blue Iris.

**Specifications:**
- 10 FPS (production) or 2 FPS (DEV_SLOW)
- Used for YOLO detection
- Saved for Armed/Active/Post cameras

**Always write as:** "sub profile" or "sub" (lowercase)

---

### Main Profile
**Definition:** High-res JPEG stream from Blue Iris.

**Specifications:**
- Captured only during Active state
- Higher quality for face/LPR/playback
- Saved alongside sub frames

**Always write as:** "main profile" or "main" (lowercase)

---

### Frame
**Definition:** Single JPEG snapshot with metadata.

**Structure:**
```python
{
  "camera_id": "FrontDoor",
  "ts_utc": "2023-11-07T12:34:56.789Z",
  "profile": "sub" | "main",
  "jpeg_bytes": b"..."
}
```

**Always write as:** "frame" (lowercase singular), "frames" (plural)

**NOT:** "image" (use "image" for processed crops, "frame" for raw data)

---

## Configuration

### Stage
**Definition:** Operational configuration preset.

**Values:**
- **DEV_SLOW** - 2 FPS, minimal cameras, no external AI
- **DEV_FULL** - 10 FPS, all cameras, all features
- **PROD_LEARNING** - Conservative merging, human review active
- **PROD_AUTO** - Aggressive merging, minimal review

**Always write as:** "Stage" (capitalized) or "stage: DEV_SLOW"

**NOT:** "mode", "profile", "environment"

---

### Phase
**Definition:** Architectural development milestone.

**Values:** Phase 0, Phase 1, Phase 2, Phase 3, Phase 4, Phase 5

**Scope:**
- Phase 0: ChunkProcessor (YOLO → LocalTracks)
- Phase 1: TemporalLinker (same-camera linking)
- Phase 2: IdentityResolver (cross-camera merging)
- Phase 3+: Refinement

**Always write as:** "Phase 0", "Phase 1" (capitalized)

**NOT:** "stage", "milestone"

---

## Storage & Retention

### Frames
**Retention:** 9 days (production), 24 hours (DEV_SLOW)

**Scope:** All sub frames for Armed/Active/Post cameras.

---

### Metadata
**Retention:** Indefinite

**Includes:**
- LocalTracks (with all relationships)
- Learned 6x6 grids
- Face recognition results
- Human corrections

---

### Image Bank
**Retention:** Indefinite

**Contents:** Top 3 quality images per agent × camera × lighting.

**Purpose:** Manual re-identification fallback.

**Selection:** `quality_score = bbox_area × detection_confidence`

**Always write as:** "image bank" (lowercase, two words)

---

## System Names

### MarengoCam / Marengo
**This System:** Complete detection/tracking/merge/archiving pipeline.

**Components:** ChunkProcessor, TemporalLinker, IdentityResolver, EvidenceProcessor

**CLI:** `marengo` command

**Always write as:** "MarengoCam" (full name) or "Marengo" (short)

---

### Blue Iris
**Role:** Camera connection manager and JPEG provider.

**What it does:**
- Manages camera connections (RTSP)
- Provides JPEG snapshots via HTTP API
- Serves `/image/{short}?q=60` endpoint

**What it does NOT do:**
- Video recording (Marengo reconstructs from JPEGs)
- Detection or tracking
- Event logic

**Always write as:** "Blue Iris" (two words) or "BI"

---

## Common Mistakes to Avoid

❌ **Wrong:** "The agent is stored in the database"
✅ **Right:** "LocalTracks are stored; agent info is queried from tracks"

❌ **Wrong:** "Track segment with duration 12s"
✅ **Right:** "LocalTrack with variable duration (up to 12s)"

❌ **Wrong:** "GlobalAgent.current_state field"
✅ **Right:** "Current state computed from latest LocalTrack"

❌ **Wrong:** "Constraint object for uncertainty"
✅ **Right:** "LocalTrack.identity_candidates for uncertainty"

❌ **Wrong:** "Camera enters Active mode"
✅ **Right:** "Camera transitions to Active state"

---

## Deprecated Terms (Do Not Use)

- ~~"Track Segment"~~ → Use "LocalTrack"
- ~~"Camera Track"~~ → Use "linked LocalTracks" or "LocalTrack chain"
- ~~"Candidate Track"~~ → Use "LocalTrack in Armed state" (implementation detail, not primary concept)
- ~~"Constraint"~~ → Use "identity_candidates on LocalTrack"
- ~~"Container"~~ → Use "vehicle occupancy" or "structure entry" (context flags, not objects)
- ~~"GlobalAgent table"~~ → Use "query LocalTracks by global_agent_id"

---

## When in Doubt

1. Check this nomenclature file
2. Check L2_DecisionArchitecture.md (architectural definitions)
3. Search existing code for usage examples
4. Prefer explicit over abbreviated
5. Maintain consistency with established patterns

**Last Updated:** 2025-11-14
