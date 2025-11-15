# Level 13 - Orchestration

## Purpose
Event-driven orchestration model. Defines what triggers what and when.

---

## Frame Acquisition (Continuous)

**Per camera:**
- Pull 10 JPEGs/second from Blue Iris HTTP endpoint
- Store in 25-second ring buffer (250 frames)
- Oldest frames drop off automatically
- **Runs continuously regardless of camera state**

**Why 25s buffer:** Provides wiggle room for 12s chunks + overlap + playback

---

## Camera State Machine (Event-Driven)

**ChunkProcessor is triggered by camera state transitions, not timers.**

### State: Standby (Quiet)
**Behavior:**
- Motion gate active (pixel diff check)
- Sparse YOLO inference
- No frame persistence (ring buffer only)

**Transition out:**
- Motion detected → **Armed**

---

### State: Armed (Confirming)
**Behavior:**
- Moderate YOLO inference (confirming sustained movement)
- Accumulate detections (candidate track)
- Save sub frames to disk

**Transition out:**
- ≥3s sustained movement → **Active** (candidate promoted to LocalTrack)
- Motion stops before 3s → **Standby** (candidate discarded)

---

### State: Active (Tracking)
**Behavior:**
- Full YOLO inference
- At least one confirmed LocalTrack being tracked
- Save sub + main frames to disk

**Transition out:**
- All tracks end (agent leaves frame / crosses portal / quiet period) → **Post**

**⚠️ TRIGGER: Active → Post transition fires chunk processing**

```python
def on_active_to_post(camera_id, transition_time):
    # Grab last 12s from ring buffer
    chunk_frames = frame_buffer.get_range(
        camera_id,
        transition_time - 12,
        transition_time
    )

    # Process chunk
    process_chunk(camera_id, chunk_frames, transition_time)
```

---

### State: Post (Cooldown)
**Behavior:**
- Moderate YOLO inference (watching for re-entry)
- Save sub frames to disk
- Duration: post_roll_s (10-15s, configured)

**Transition out:**
- New detection → **Active** (agent returned)
- Post-roll expires → **Standby**

---

## Chunk Processing Pipeline

**Triggered by:** Active → Post transition

```python
def process_chunk(camera_id, chunk_frames, end_time):
    # 1. ChunkProcessor creates LocalTracks
    #    - Runs YOLO on 12s of frames
    #    - Creates one LocalTrack per YOLO tracking ID
    #    - Track duration is variable (up to 12s)
    new_tracks = ChunkProcessor.process(camera_id, chunk_frames, end_time)

    if not new_tracks:
        return  # No tracks detected in chunk

    # 2. TemporalLinker connects across chunk boundary
    #    - Uses 2s overlap zone from previous chunk
    #    - Matches by position + appearance in overlap
    #    - Updates previous_track/next_track pointers
    #    - Propagates identity if previous track had it
    previous_chunk = get_previous_chunk_tracks(camera_id, end_time - 10)
    TemporalLinker.link(camera_id, new_tracks, previous_chunk)

    # 3. Save to database
    db.save_tracks(new_tracks)

    # 4. IdentityResolver attempts cross-camera merges
    #    - For each new track without global_agent_id
    #    - Evidence hierarchy: face → grid overlap → grid+alibi → queue
    for track in new_tracks:
        if track.global_agent_id is None:
            IdentityResolver.resolve(track)
```

---

## Chunk Overlap Explanation

**When there's continuous activity:**

```
Track ends at T=10  → Active→Post → process chunk [T=-2 to T=10]
Track ends at T=20  → Active→Post → process chunk [T=8 to T=20]
Track ends at T=30  → Active→Post → process chunk [T=18 to T=30]
                                     ^^2s overlap^^

Overlap zone (T=8-10 in second chunk) used by TemporalLinker
```

**Key insight:** "Every 10s" from L3 docs is typical frequency when agents move through FOV continuously, not a timer.

**When activity is brief:**

```
Agent visible 5s → Active→Post → process chunk [T=-7 to T=5]
(Only one chunk, track duration = 5s)
```

---

## Track Duration (Variable)

**LocalTracks are NOT fixed 10s or 12s duration.**

Track duration determined by:
- **Agent leaves frame:** Track ends when agent exits FOV
  - `exit_reason = "left_frame"`
  - `next_track = None`

- **Chunk boundary (agent still visible):** Track continues in next chunk
  - `exit_reason = "chunk_boundary"`
  - `next_track` set by TemporalLinker to next chunk's track

- **Portal crossed:** Agent enters structure/vehicle
  - `exit_reason = "portal_crossed"`
  - `at_portal = "garage_door"`

**Examples:**
- Agent walks through FOV in 7s → track duration = 7s
- Agent visible across 3 chunks → 3 linked tracks (12s, 10s, 4s)
- Agent barely triggers Armed, leaves quickly → no track (discarded <3s)

---

## Evidence Loop (Event-Driven)

**Triggered by:**
- Face recognition result from CodeProject.AI (async HTTP callback)
- Human review submission from GUI (HTTP POST)
- LLM decision (immediate, no batching)

```python
def process_evidence(evidence):
    # 1. EvidenceProcessor resolves track
    #    - Sets global_agent_id or removes from identity_candidates
    resolved_tracks = EvidenceProcessor.process(evidence)

    # 2. Cascade updates
    #    - Propagate through previous_track/next_track chains
    #    - Apply process of elimination to concurrent uncertain tracks
    for track in resolved_tracks:
        EvidenceProcessor.propagate(track)
        eliminated_tracks = EvidenceProcessor.eliminate_alternatives(track)

    # 3. Re-run IdentityResolver on affected uncertain tracks
    #    - Process of elimination might leave single candidate → auto-resolve
    #    - Newly resolved tracks might enable more merges (cascade)
    for track in eliminated_tracks:
        if len(track.identity_candidates) == 1:
            # Only one candidate left → resolve
            track.global_agent_id = list(track.identity_candidates)[0]
            IdentityResolver.update_grid_learning(track)  # Learn from resolution
        elif len(track.identity_candidates) > 1:
            # Still uncertain, try merge again with new information
            IdentityResolver.resolve(track)
```

**Why re-run IdentityResolver:**

Example cascade:
```
Track A (Cam1, t=10) → candidates: {Agent_1, Agent_2}
Track B (Cam2, t=15) → candidates: {Agent_1, Agent_2}
Track C (Cam3, t=20) → candidates: {Agent_1, Agent_2}

Human reviews Track A → Agent_1

EvidenceProcessor:
  1. Track A.global_agent_id = Agent_1
  2. Process of elimination:
     - Track B.candidates.remove(Agent_1) → {Agent_2} → AUTO-RESOLVE
     - Track C.candidates.remove(Agent_1) → {Agent_2} → AUTO-RESOLVE
  3. IdentityResolver updates grid learning from resolved tracks
```

---

## Component Interactions

### ChunkProcessor
**Triggered by:** Active→Post state transition
**Calls:** YOLO11 inference
**Manages:** Camera state machine internally
**Returns:** List[LocalTrack] (variable duration tracks)

### TemporalLinker
**Triggered by:** After ChunkProcessor
**Calls:** Database (read previous chunk tracks, update pointers)
**Uses:** 2s overlap zone for matching

### IdentityResolver
**Triggered by:**
- After TemporalLinker (for new unlinked tracks)
- After EvidenceProcessor (after process of elimination)
**Calls:**
- CodeProject.AI (face recognition)
- Database (grid queries, alibi checks)
- Review queue (if uncertain)

### EvidenceProcessor
**Triggered by:** Evidence arrival (async events)
**Calls:**
- Database (read/write tracks)
- IdentityResolver (re-attempt merges after updates)

### GUI
**Triggered by:** User actions (HTTP requests)
**Calls:** Database (queries only)
**Updates:** Server-Sent Events (SSE) push to connected clients when tracks resolve

### Configuration
**Loaded:** On startup, hot-reloaded on file change
**Used by:** All components (read-only)

### SystemHealth
**Runs:** Separate monitoring thread (every 5s)
**Calls:** Database (query metrics), Alert channels

---

## Data Flow Summary

```
Frame Buffer (continuous, 10 FPS per camera)
    ↓
[Camera State Machine] monitors motion/detections
    ↓
Active→Post transition triggers ↓
    ↓
[ChunkProcessor] 12s chunk → LocalTracks (variable duration)
    ↓
[TemporalLinker] → LocalTracks (linked same-camera via 2s overlap)
    ↓
[IdentityResolver] → LocalTracks (global_agent_id OR identity_candidates)
    ↓
[Database] → stored
    ↓
[GUI] → queries on demand, SSE updates

[Evidence arrives] → [EvidenceProcessor] → resolve + cascade → [IdentityResolver]
```

---

## Concurrency Model

**Frame acquisition:** One thread per camera (blocking HTTP pulls every 100ms)

**State machine:** One thread per camera (monitors motion, triggers chunk processing)

**Chunk processing:** Triggered events (can run in parallel per camera)

**Evidence loop:** Single async queue (process serially to avoid race conditions on identity_candidates)

**Database:** PostgreSQL handles concurrency (ACID transactions)

---

## Error Handling

**Frame fetch fails:** Log warning, continue (ring buffer retains old frames)

**YOLO inference fails:** Log error, skip chunk, return to previous state

**Database write fails:** Rollback transaction, retry once, log critical alert

**External AI timeout:** Mark track uncertain, queue for review

**State machine stuck:** Watchdog timer resets camera to Standby after 60s

---

## Bootstrap Phase Behavior

**Online Learning Mode (initial 2-4 weeks):**
- IdentityResolver: Conservative thresholds (face ≥ 0.85)
- All uncertain tracks → review queue
- Human validates all face library additions

**Autonomous Mode (after bootstrap):**
- IdentityResolver: Lower thresholds (face ≥ 0.70)
- Auto-merge enabled
- LLM handles most uncertain tracks

**Transition criteria:** Auto-merge rate > 80%, grid > 80% learned

**See:** L3_Configuration.md for operational modes
