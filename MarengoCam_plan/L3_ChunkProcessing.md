# Level 3 - Chunk Processing (ChunkProcessor)

## Purpose
Convert 12-second video chunks into LocalTracks.

---

## Input/Output

**Input:** 12s video chunk from one camera (sub profile JPEG stream)

**Output:** List of LocalTracks (one per YOLO tracking ID)

**Timing:** Every 10s (overlapping: T=0-12s, T=10-22s, T=20-32s...)

**Overlap:** 2s overlap allows TemporalLinker to match tracks across chunks.

---

## Motion Gate (Standby Cameras Only)

**Purpose:** Fast pixel-diff check before running YOLO.

**How:**
1. Compare current frame to previous frame
2. If motion delta < threshold → skip YOLO
3. If motion delta ≥ threshold → proceed to YOLO

**Filters:** Static frames, minimal jitter, lighting changes below threshold.

**Result:** Only frames with motion proceed to YOLO (saves GPU cycles).

---

## YOLO11 Tracking

**Detector:** YOLO11 with built-in tracking
- Classes: person, vehicle, animal
- Confidence threshold filters low-quality detections
- Runs on sub profile frames (10 FPS)

**Adaptive Inference Rate by Camera State:**

**Standby:** Sparse sampling (motion gate active), no frame persistence

**Armed:** Moderate sampling (confirming track), sub frames saved

**Active:** Full inference (all frames within GPU budget), sub + main frames saved

**Post:** Moderate sampling (watching re-entry), sub frames saved

---

## Track Quality Filtering

**Sustained Movement Requirement (≥3s):**
1. Motion gate triggers → Standby → Armed
2. Armed → accumulate detections (candidate track)
3. If sustained ≥3s → promote to LocalTrack, Armed → Active
4. If motion stops <3s → discard, Armed → Standby

**Filters:** Brief glitches, detector noise, fleeting shadows.

**Confidence threshold:**
- Person: 0.5
- Vehicle: 0.6
- Animal: 0.4

---

## Output: LocalTracks

```python
LocalTrack(
    local_id=yolo_tracking_id,  # Chunk-scoped
    camera_id="FrontDoor",
    chunk_id=42,
    start_time=1699363200.0,
    end_time=1699363207.5,  # Variable (agent may leave early)
    entry_cell=(3, 2),  # 6x6 grid
    exit_cell=(4, 3),
    detections=[...],
    exit_reason="left_frame",  # or "chunk_boundary" or "portal_crossed"
    previous_track=None,  # Set by TemporalLinker
    next_track=None,
    global_agent_id=None,  # Set by IdentityResolver
    identity_candidates=set(),
    decision_record=None  # JSON blob detailing the IdentityResolver's decision and evidence
)
```

---

## Camera State Transitions

**Standby → Armed:** Motion gate + sparse YOLO detect object

**Armed → Active:** Candidate track promoted (≥3s sustained movement)

**Active → Post:** All LocalTracks ended (quiet period)

**Post → Standby:** Post-roll expires (no new detections)

**Note:** All cameras polled continuously at 10 FPS. States control inference rate and frame persistence.

---

## Portal Association

**Purpose:** Identify when a track originates from or terminates at a known portal, including dynamically identified stationary vehicles.

**Process:**
1.  **Load Static Portals:** Load fixed portal locations (doors, garages) from `marengo.yaml`.
2.  **Identify Dynamic Portals:** After chunk processing, identify `vehicle` tracks that have been stationary for a significant duration (e.g., > 10 seconds across one or more chunks). These are treated as temporary "dynamic portals." Their portal ID is derived from their track ID (e.g., `vehicle_123`).
3.  **Associate Tracks:** For each `LocalTrack` in the chunk:
    - If the track's first appearance (`entry_cell`) is within the cells of a static or dynamic portal, set `track.at_portal = portal_id`.
    - If the track's last appearance (`exit_cell`) is within the cells of a portal, update `track.exit_reason = "portal_crossed"`.

**Example:** A `person` track appears next to a car that has been parked for 5 minutes. The `ChunkProcessor` identifies the car as a dynamic portal and sets the new person track's `at_portal` field, providing crucial context for the `IdentityResolver`.
