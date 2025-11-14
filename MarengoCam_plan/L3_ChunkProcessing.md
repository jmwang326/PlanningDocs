# Level 3 - Chunk Processing (ChunkProcessor)

## Purpose
Tactical guide for converting 12-second video chunks into LocalTracks. This is the first handler in the pipeline.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for ChunkProcessor architecture overview.

---

## Input/Output

**Input:** 12-second video chunk from one camera (sub profile JPEG stream)

**Output:** List of LocalTracks (one per YOLO tracking ID)

**Timing:** Every 10 seconds (overlapping chunks: T=0-12s, T=10-22s, T=20-32s...)

---

## Chunked Processing Strategy

### Why 12-Second Chunks with 2-Second Overlap

**Chunk duration:** 12 seconds (fixed)
- Real-time processing: keeps up with incoming video
- Multiple streams: pipeline chunks across 10+ cameras
- YOLO inference batching: efficient GPU utilization

**Overlap:** 2 seconds (last 2s of chunk N = first 2s of chunk N+1)
- YOLO tracking IDs reset each chunk
- Overlap allows TemporalLinker to match "Local ID 3 in chunk N" = "Local ID 7 in chunk N+1"
- Ensures continuity despite YOLO ID discontinuity

---

## Filtering: Motion Gate

### Purpose
Fast pixel-diff check before running YOLO to filter static frames.

### When Active
Standby cameras only (Armed/Active/Post cameras skip motion gate, run YOLO directly)

### How It Works
1. Compare current frame to previous frame (pixel-level difference)
2. If motion delta < threshold → skip YOLO inference (static frame)
3. If motion delta ≥ threshold → proceed to sparse YOLO sampling

### What Gets Filtered
- Static frames (trees, parked cars, shadows)
- Minimal camera jitter
- Lighting changes below threshold

**Result:** Only frames with motion proceed to YOLO, saving GPU cycles.

**See:** L13_MotionGate.md for implementation details (pixel-diff algorithm, thresholds)

---

## Detection: YOLO11 Tracking

### Detector
**YOLO11** with built-in tracking
- Classes: person, vehicle, animal
- Confidence threshold filters low-quality detections
- Runs on sub profile frames (10 FPS)

### Adaptive Inference Rate by Camera State

**Standby:**
- Sparse YOLO sampling (motion gate filters most frames)
- Minimal GPU allocation
- Frame persistence: none (25s buffer only)

**Armed:**
- Moderate YOLO sampling (confirming sustained track)
- Increased GPU allocation
- Frame persistence: sub frames saved to disk

**Active:**
- Full YOLO inference (all frames within GPU budget)
- Maximum GPU allocation
- Frame persistence: sub + main frames saved

**Post:**
- Moderate YOLO sampling (watching for re-entry)
- Reduced GPU allocation
- Frame persistence: sub frames saved

**Note:** Inference Manager determines actual sampling rates based on GPU budget. Prioritizes Active cameras (~3 Hz sustained), allocates remaining to Armed/Post, minimal to Standby.

**See:** L13_InferenceManager.md for GPU scheduling details

---

## Track Quality Filtering

### Sustained Movement Requirement (≥3s)

**Candidate Track Lifecycle:**
1. Motion gate triggers → camera Standby → Armed
2. Armed state → accumulate detections (candidate track)
3. If sustained movement ≥3s → promote to LocalTrack, camera → Active
4. If motion stops < 3s → discard candidate, camera → Standby

**Why this matters:** Filters brief glitches, detector noise, fleeting shadows.

**What gets filtered:**
- Brief detections (< 3s duration)
- Stationary objects (parked car detected once, doesn't move)
- Detector glitches (single-frame false positive)

**See:** L13_TrackStateManager.md for candidate track mechanics

---

### Confidence Threshold

Only detections with YOLO confidence ≥ threshold create tracks.

**Configurable per class:**
- Person: 0.5 (default)
- Vehicle: 0.6 (default)
- Animal: 0.4 (default)

**See:** L3_Configuration.md for threshold settings

---

## Output: LocalTracks

### What Gets Created

For each YOLO tracking ID in the chunk that meets quality filters:

```python
LocalTrack(
    local_id=yolo_tracking_id,  # Chunk-scoped
    camera_id="FrontDoor",
    chunk_id=42,
    start_time=1699363200.0,
    end_time=1699363207.5,  # Variable duration (agent may leave early)
    entry_cell=(3, 2),  # 6x6 grid
    exit_cell=(4, 3),
    detections=[...],  # Frame-by-frame bboxes
    exit_reason="left_frame",  # or "chunk_boundary"
    # Relationships set by TemporalLinker
    previous_track=None,
    next_track=None,
    # Identity set by IdentityResolver
    global_agent_id=None,
    identity_candidates=set()
)
```

### Track Duration

**Tracks are variable length (not always 12s):**
- Agent may enter mid-chunk (start_time > chunk_start)
- Agent may leave mid-chunk (end_time < chunk_end)
- exit_reason indicates why track ended:
  - `"left_frame"` → agent left camera FOV
  - `"chunk_boundary"` → track continues in next chunk
  - `"portal_crossed"` → agent crossed configured portal

---

## Camera State Transitions

### Standby → Armed
**Trigger:** Motion gate + sparse YOLO detect object
**Behavior:** Increase inference rate, begin saving frames

### Armed → Active
**Trigger:** Candidate track promoted (≥3s sustained movement)
**Behavior:** Full inference, save sub + main frames, create LocalTrack

### Active → Post
**Trigger:** All LocalTracks ended (no detections for quiet period)
**Behavior:** Continue saving through post-roll

### Post → Standby
**Trigger:** Post-roll expires (no new detections)
**Behavior:** Return to sparse sampling, stop frame persistence

**Note:** All cameras polled continuously at 10 FPS. States control inference rate and frame persistence, not acquisition.

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - ChunkProcessor overview
- **L2_Strategy.md** - Why filter noise first

### Next Handler
- **L3_TemporalLinking.md** - Links LocalTracks across chunks

### Implementation
- **L13_MotionGate.md** - Pixel-diff algorithm
- **L13_InferenceManager.md** - GPU scheduling, sampling rates
- **L13_TrackStateManager.md** - Candidate track mechanics, state machine
- **L13_Detection.md** - LocalTrack data structure
