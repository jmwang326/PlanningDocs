# MarengoCam Nomenclature Guide

**Purpose:** Define standard terminology used throughout the project to ensure consistency across code, documentation, and communication.

**Last Updated:** November 7, 2025

---

## Core Entities

### Agent
**What it is:** A persistent entity (person, vehicle, or animal) tracked across cameras and time.

**What it's NOT:** Just a detection or a track segment.

**Location States (Person):**
- `visible` - Actively tracked walking/moving on camera
- `visible_in_vehicle(V)` - Sporadic detections through vehicle windows (fragmented tracks)
- `offscreen` - Between cameras (expected portal transition, time-bounded)
- `in_structure(XYZ)` - Inside building/garage via portal (inferred)
- `in_vehicle(V)` - Inside vehicle, not visible (fully occluded)
- `unknown` - Lost track, location unclear
- `exited` - Crossed property exit portal, confirmed departure

**Location States (Vehicle):**
- `visible_parked` - Tracked on camera, stationary (boring, camera stays Armed)
- `visible_moving` - Tracked on camera, in motion (interesting, Active tracking)
- `offscreen` - Between cameras (expected portal transition, time-bounded)
- `in_structure(XYZ)` - Inside garage via portal (inferred)
- `unknown` - Lost track, location unclear
- `exited` - Crossed property exit portal, confirmed departure
- `occupied_by([])` - List of person agents inside (with uncertainty)

**Key Behaviors:**
- All cameras outdoor: `in_structure` always inferred from portal crossing
- Person in vehicle (either state) exempt from cross-camera person merge rules
- Vehicle movement state affects camera behavior (parked=boring, moving=interesting)

**Key Properties:**
- `track_segments[]` - All camera-observable windows
- `current_state` - Visible/Hidden with location and confidence
- `image_bank{}` - Best quality images per camera × lighting
- `face_id` - CodeProject.AI face library ID (once registered)
- `merge_history[]` - All merge decisions with evidence

**Identity Progression:**
- Starts as `Unknown-N` (sequential numbering)
- Promoted via face recognition (confidence ≥ [Face Recognition Threshold]) or validated merge
- Never deleted, only terminated

**Usage Examples:**
```python
agent.id = "A_20231107_001"  # agent_ prefix
agent.type = "person" | "vehicle" | "animal"
agent.current_state = {'visible': True, 'camera': 'FrontDoor', 'cell_6x6': (3,4)}
```

---

### Track Segment
**What it is:** One continuous visibility window of an agent on a single camera.

**Contains:**
- `camera` - Camera ID where segment observed
- `start_time`, `end_time` - UTC timestamps
- `entry_cell_6x6`, `exit_cell_6x6` - Where agent entered/exited frame, in 6x6 grid coordinates.
- `detections[]` - Frame-by-frame bounding boxes and confidences from the YOLO tracker.
- `quality_score` - For image selection (bbox_area × detection_confidence)

**Relationship:** Multiple track segments can belong to one agent after merging.

**Usage Examples:**
```python
segment.id = "TS_FrontDoor_20231107_120532"
segment.agent_id = "A_20231107_001"  # Links to parent agent
segment.duration = end_time - start_time  # Seconds
```

**NOT called:** "track", "segment", "observation" (alone)

---

### Detection
**What it is:** Single YOLO inference result for one frame, one bounding box.

**Contains:**
- `timestamp` - UTC
- `bbox` - (x, y, width, height) in pixels
- `class` - "person" | "vehicle" | "animal"
- `confidence` - YOLO detection confidence [0.0, 1.0]
- `cell_6x6` - Grid cell of the 6x6 grid containing the bbox centroid.

**Relationship:** Multiple detections (frame-to-frame) form a track segment.

**Usage Examples:**
```python
detection.class = "person"  # Lowercase
detection.confidence = 0.87
detection.cell_32x32 = (12, 18)  # Row, col
```

**NOT called:** "inference", "result", "bbox" (use "detection" as noun)

---

## Grid Systems

### 6x6 Inter-Camera Adjacency Grid
**Purpose:** Learn and store the minimum travel time between cells of a camera pair to enable plausible track merging.

**Specifications:**
- **Matrix:** A `6x6` matrix is maintained for each directed camera pair and for each agent class (e.g., `person`, `vehicle`).
- **Cell Value:** Stores the minimum observed travel time in milliseconds between a source camera's exit cell and a destination camera's entry cell.
- **Overlap:** A time of `0` indicates a direct camera overlap zone.
- **Unlearned:** A value of `-1` (or `MAX_INT`) represents an unlearned path.

**Learning:**
- The grid is populated and refined over time by recording the time deltas from high-confidence, validated merges. The system always stores the *lowest* observed time.

**Usage Examples:**
```python
min_travel_time = grid.get_time('FrontDoor', 'Driveway', 'person', (5, 2), (0, 3))
if min_travel_time == 0:
    # This is an overlap zone
```

**Always write as:** "6x6 grid" or "inter-camera grid"

**NOT:** "32x32 grid", "8x8 grid", "adjacency matrix"

---

## Camera States (Agent-Driven State Machine)

### Standby
**Definition:** Quiet baseline, no recent activity.

**Behavior:**
- Acquisition: 10 FPS (sub profile only)
- Inference: Sparse (~10% sampling after motion gate)
- Frame persistence: No (pre-roll ring buffer only)
- Main profile: Not captured

**Transitions:**
- → Armed: Motion detected or neighbor Active
- ← Post: Post-roll expires without new activity

**Always write as:** "Standby" (capitalized)

**NOT:** "Idle", "Quiet", "Inactive"

---

### Armed
**Definition:** Recent activity or neighbor camera is Active.

**Behavior:**
- Acquisition: 10 FPS (sub profile)
- Inference: Medium (~60% sampling)
- Frame persistence: Yes (all sub frames saved)
- Main profile: Not captured

**Triggers:**
- Neighbor boost (adjacent camera Active via learned 8×8 edges)
- Recent detection (single high-confidence or motion spike)

**Transitions:**
- → Active: Agent confirmed (≥3s sustained movement)
- → Standby: Expires after arm_expire_s without detections

**Always write as:** "Armed" (capitalized)

**NOT:** "Alerted", "Alert", "Pre-Active", "Armed/Alerted"

**Note:** Some older docs use "Alerted" — update to "Armed" for consistency.

---

### Active
**Definition:** At least one persistent agent track is being reported by the YOLO tracker.

**Behavior:**
- Acquisition: 10 FPS (sub + main profiles)
- Inference: High (handled by tracker)
- Frame persistence: Yes (all sub + main frames saved)
- Main profile: Captured for Active cameras

**Transitions:**
- → Post: YOLO tracker has lost all tracks for `[quiet_window_s]`.

**Always write as:** "Active" (capitalized)

**NOT:** "Recording", "Event", "Triggered"

---

### Post
**Definition:** Cool-down after agent leaves frame.

**Behavior:**
- Acquisition: 10 FPS (sub profile only)
- Inference: Medium (~40% sampling)
- Frame persistence: Yes (all sub frames saved)
- Duration: post_roll_s (10-15 seconds, TBD)

**Purpose:** Capture post-event context, allow re-activation if agent returns.

**Transitions:**
- → Active: New detection during Post
- → Standby: Post-roll expires without detections

**Always write as:** "Post" (capitalized)

**NOT:** "Post-Active", "Cooldown", "Post-Event"

---

### Suppressed
**Definition:** System pressure override (global, not per-agent).

**Behavior:**
- Acquisition: Varies (may skip frames)
- Inference: Scaled back for non-Active cameras
- Active cameras: Maintain fidelity (always 100% inference)
- Purpose: Protect system during high load

**Triggers:**
- GPU saturation (queue depth > D_max)
- Blue Iris error storm
- Global AI budget exceeded

**Transitions:**
- ← Any state: Pressure detected
- → Prior state: Pressure cleared

**Always write as:** "Suppressed" (capitalized)

**NOT:** "Throttled", "Degraded", "Overload"

---

## Merge Decision Hierarchy

### Auto-Merge
**Definition:** Automatic merge without human/LLM review.

**Triggers:**
- Plausible transition according to the `6x6` grid with no other competing candidates.
- High-confidence face match (≥ [Face Recognition Threshold]).
- License plate match (vehicles).

**Outcome:** Track segments immediately combined into single agent.

**Always write as:** "Auto-Merge" (hyphenated, capitalized)

---

### Queue for Review
**Definition:** Plausible merge candidate, needs human or LLM adjudication.

**Triggers:**
- Time/space plausible (within learned travel window)
- No strong identity evidence (face uncertain, no plates)
- Appearance similar but not definitive

**Process:**
- Generate comparison panels (side-by-side images)
- Present to LLM with context (cameras, cells, times)
- Human review if LLM returns CANNOT

**Always write as:** "Queue for Review" or "queued"

**NOT:** "Manual review", "pending", "deferred"

---

### Reject
**Definition:** Not the same agent (definitive).

**Evidence:**
- Face mismatch (different identities)
- Appearance conflict (clothing, color)
- Spatial/temporal impossibility

**Outcome:** Track segments remain separate agents.

**Always write as:** "Reject" (capitalized)

**NOT:** "Deny", "Refuse", "Negative"

---

### CANNOT
**Definition:** LLM cannot make confident decision.

**Behavior:**
- Mark candidate as provisional
- Exclude from aggregate updates (edge learning)
- May re-prompt with more context later

**LLM Response Format:**
```json
{
  "decision": "CANNOT",
  "reasoning": "Insufficient visibility due to occlusion"
}
```

**Always write as:** "CANNOT" (all uppercase)

**NOT:** "Unknown", "Uncertain", "Undecided"

---

## External AI Services

### CodeProject.AI
**What it is:** External AI inference service for specialized models.

**Services:**
- `face_recognition` - Trainable face library with name registration
- `lpr` - License Plate Recognition

**Multi-Instance Support:**
- Multiple servers can run simultaneously (different machines)
- Priority routing (priority 1 tried first, failover to priority 2)
- Health tracking per service (response times, queue depth, success rate)

**Always write as:** "CodeProject.AI"

**NOT:** "CPAI", "CP.AI", "CodeProject", "CPAi"

---

### Face Recognition
**Method:** CodeProject.AI trainable library.

**Workflow:**
1. Agent confirmed via merge
2. Best face crops extracted from image bank
3. Registered in CodeProject.AI with name
4. Future detections auto-recognized (confidence ≥0.90 → auto-merge)

**Image Quality Metric:** `bbox_area × detection_confidence` (simple, works)

**Storage:**
- **Face library:** CodeProject.AI trainable database (indefinite retention)
- **Image bank:** Top 3 quality images per agent × camera × lighting (indefinite)

**Always write as:** "face recognition" (lowercase)

**NOT:** "facial recognition", "FR", "face match" (use for result, not service)

---

### LPR (License Plate Recognition)
**Method:** CodeProject.AI model.

**Priority:** Higher than face for vehicles (plates more stable across lighting).

**Always write as:** "LPR" (uppercase acronym)

**NOT:** "license plate recognition" (spell out only in definitions), "plate recognition"

---

## Frame Terminology

### Sub Profile
**Definition:** Low-res JPEG stream from Blue Iris.

**Specifications:**
- Always available at 10 FPS (production) or 2 FPS (DEV_SLOW)
- Used for YOLO detection
- Saved for Armed/Active/Post cameras
- Sufficient quality for detection and tracking

**Always write as:** "sub profile" or "sub" (lowercase)

**NOT:** "substream", "low-res stream", "detection stream"

---

### Main Profile
**Definition:** High-res JPEG stream from Blue Iris.

**Specifications:**
- Captured only during Active state
- Higher quality for face recognition, LPR, playback
- Saved alongside sub frames for Active cameras
- Neighbor policy TBD (may capture for adjacent Active cameras)

**Always write as:** "main profile" or "main" (lowercase)

**NOT:** "mainstream", "high-res stream", "detail stream"

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

**Frame Rate:**
- Production: 10 FPS (Fixed)
- DEV_SLOW: 2 FPS

**Always write as:** "frame" (lowercase, singular) or "frames" (plural)

**NOT:** "image" (use "image" for processed crops/panels, "frame" for pipeline data)

---

## Configuration & Operations

### Stage (vs Phase)
**Definition:** Operational configuration preset controlling resource usage.

**Values:**
- **DEV_SLOW** - 2 FPS, 5 cameras, no merging, no external AI (underpowered laptop)
- **DEV_FULL** - 10 FPS, all cameras, merging enabled, face/LPR active
- **PROD_LEARNING** - Conservative auto-merge 0.85, human review queue active
- **PROD_AUTO** - Aggressive auto-merge 0.70, minimal review

**NOT the same as:** Phase (architectural development milestone: Phase 0-5)

**Always write as:** "Stage" (capitalized) or "stage: DEV_SLOW" (config context)

**NOT:** "mode", "profile", "environment"

---

### Phase (vs Stage)
**Definition:** Architectural development milestone (project planning).

**Values:** Phase 0, Phase 1, Phase 2, Phase 3, Phase 4, Phase 5

**Scope:**
- Phase 0: Camera acquisition + motion gate
- Phase 1: Detection + intra-camera tracking
- Phase 2: Cross-camera merging (time/space + LLM)
- Phase 3: Face library + edge visualization
- Phase 4: Timeline viewer + event reconstruction
- Phase 5: Full adjacency learning + polish

**NOT the same as:** Stage (operational config preset)

**Always write as:** "Phase 0", "Phase 1", etc. (capitalized)

**NOT:** "stage", "milestone", "sprint"

---

### Hot-Reload
**Definition:** Apply configuration changes without restarting services.

**Mechanism:**
- Watch config files for changes.
- Validate schema before applying.
- Dry-run with auto-rollback if errors spike.
- Version stamping on all artifacts.

**Always write as:** "hot-reload" (hyphenated, lowercase)

**NOT:** "live reload", "dynamic config", "hot swap"

---

## Time Windows & Thresholds

### Travel Time
**Definitions:**
- **min_time** - The minimum observed travel time (Δt) between two cells in the `6x6` grid.

**Usage:** Gating merge candidates, neighbor boost timing.

**Always write as:** "travel time" or "Δt" (delta-t)

**NOT:** "transition time", "transit time", "duration"

---

### Pre-roll / Post-roll
**Definitions:**
- **Pre-roll:** Ring buffer before Armed (7 seconds)
- **Post-roll:** Persistence after Active ends (10-15 seconds, TBD)

**Purpose:** Capture context before/after event for playback.

**Always write as:** "pre-roll" / "post-roll" (hyphenated, lowercase)

**NOT:** "preroll", "pre roll", "post_roll"

---

## Storage & Retention

### Frames
**Retention:**
- Production: 9 days
- DEV_SLOW: 24 hours

**Scope:** All sub frames (10 FPS) for Armed/Active/Post cameras.

**Always write as:** "frames" (lowercase, plural)

---

### Metadata
**Retention:** Indefinite

**Includes:**
- Agent tracks and merge decisions
- Learned edges (`6x6` inter-camera grid)
- Height calibration data
- Face recognition results
- Human corrections

**Always write as:** "metadata" (lowercase, one word)

**NOT:** "meta-data", "meta data", "Metadata"

---

### Image Bank
**Retention:** Indefinite

**Contents:** Top 3 quality images per agent × camera × lighting condition.

**Purpose:** Manual re-identification fallback if face recognition fails.

**Selection:** `quality_score = bbox_area × detection_confidence`

**Always write as:** "image bank" (lowercase, two words)

**NOT:** "image gallery", "image library", "imagebank"

---

### Grid Snapshots
**Retention:** 12 weeks rolling

**Contents:**
- Learned `6x6` inter-camera adjacency grids.
- Height calibration data.

**Frequency:** Weekly automated backups

**Purpose:** Recovery, visualization, and debugging of adjacency learning.

**Always write as:** "grid snapshots" (lowercase)

**NOT:** "grid backups", "snapshots", "grid state"

---

## System Names

### MarengoCam / Marengo
**This System:** Complete detection/tracking/merge/archiving pipeline.

**Components:**
- Acquisition (Blue Iris integration)
- Motion gate
- Inference scheduler
- Detector (YOLO with tracking)
- Intra-camera tracking (handled by YOLO tracker)
- Cross-camera merging (`6x6` grid)
- Archiver
- Timeline viewer

**CLI:** `marengo` command

**Always write as:** "MarengoCam" (full name) or "Marengo" (short)

**NOT:** "MarengoWatcher" (old Phase 1 name), "marengo-cam"

---

### Blue Iris
**Role:** Camera connection manager and JPEG provider.

**What it does:**
- Manages camera connections (RTSP/HTTP)
- Provides JPEG snapshots (main/sub profiles)
- Serves `/image/{short}?q=60` endpoint

**What it does NOT do:**
- Video/MP4 recording (Marengo reconstructs from JPEGs)
- Detection or tracking
- Event logic

**Always write as:** "Blue Iris" (two words, capitalized) or "BI" (abbreviation)

**NOT:** "BlueIris", "blue iris", "Blue-Iris"

---

## Abbreviations & Acronyms

**Approved:**
- **BI** - Blue Iris
- **LPR** - License Plate Recognition
- **FPS** - Frames Per Second
- **YOLO** - You Only Look Once (object detector)
- **IoU** - Intersection over Union
- **TBD** - To Be Determined (thresholds pending validation)
- **AOI** - Area of Interest (future feature)

**NOT approved (spell out):**
- CPAI (use "CodeProject.AI")
- FR (use "face recognition")
- TS (use "track segment")
- Det (use "detection")

---

## Formatting Conventions

### Code/Variables
- `agent.id` - dot notation for properties
- `cell_6x6 = (row, col)` - underscore for multi-word variables
- `face_recognition` - lowercase for service names
- `CANNOT` - uppercase for specific LLM responses

### Documentation
- **Bold** for emphasis or definitions
- `code style` for variables, functions, values
- "Quotes" for literal strings or UI text
- Capitalized states: Standby, Armed, Active, Post, Suppressed

### Naming Patterns
- Agent IDs: `A_20231107_001`
- Track Segment IDs: `TS_FrontDoor_20231107_120532`
- Camera names: `FrontDoor`, `BackYard`, `Garage` (no spaces)
- Grid cells: `(row, col)` tuple notation

---

## Common Mistakes to Avoid

❌ **Wrong:** "The detection tracked across cameras"
✅ **Right:** "The agent was tracked across cameras via multiple track segments"

❌ **Wrong:** "Camera enters Active mode"
✅ **Right:** "Camera transitions to Active state"

❌ **Wrong:** "32x32 grid" or "8x8 grid"
✅ **Right:** "6x6 inter-camera grid"

❌ **Wrong:** "Alerted state"
✅ **Right:** "Armed state"

❌ **Wrong:** "CodeProject AI service"
✅ **Right:** "CodeProject.AI service"

❌ **Wrong:** "sub stream"
✅ **Right:** "sub profile"

❌ **Wrong:** "Stage 0, Stage 1"
✅ **Right:** "Phase 0, Phase 1" (or "stage: DEV_SLOW")

---

## When in Doubt

1. Check this nomenclature file
2. Search existing code for usage examples
3. Prefer explicit over abbreviated
4. Maintain consistency with established patterns

**Last Updated:** November 7, 2025
