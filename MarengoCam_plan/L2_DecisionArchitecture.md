# Level 2 - Decision Architecture

## Purpose
Define the fundamental architecture of MarengoCam: what gets processed, who processes it, and when. This is the control flow of the entire system.

## Core Philosophy: Tracks + Handlers

**The entire system exists to manage video clips (tracks).**

Everything else - agents, timelines, uncertainty, vehicles, structures - is just metadata and queries on top of tracks.

**Two components:**
1. **Tracks** - 12-second video observations (data)
2. **Handlers** - Processors that create, link, and label tracks (logic)

That's it.

---

## The Fundamental Unit: LocalTrack

```python
class LocalTrack:
    """
    One observation from YOLO tracking (up to 12 seconds).
    Duration varies: agent may leave frame before chunk ends.
    This is the atom of the system. Everything builds from this.
    """
    # Identity within chunk
    local_id: int              # YOLO's tracking ID (chunk-scoped)
    camera_id: str
    chunk_id: int

    # Observations (what we saw)
    detections: List[Detection]  # Frame-by-frame bboxes
    start_time: float
    end_time: float            # May be < chunk_end_time if agent left early
    entry_cell: Tuple[int, int]  # 6x6 grid
    exit_cell: Tuple[int, int]

    # Relationships to other tracks
    previous_track: Optional[Tuple[int, int]]  # Same camera, previous chunk (None if first appearance)
    next_track: Optional[Tuple[int, int]]      # Same camera, next chunk (None if agent left/ended)
    merged_tracks: List[Tuple[str, int, int]]  # Other cameras, same agent
    concurrent_tracks: List[int]               # Same camera, different agents

    # Identity (resolved or uncertain)
    global_agent_id: Optional[int]      # Which agent (if known)
    identity_candidates: Set[int]       # Could be any of these (if uncertain)

    # Context flags (special circumstances)
    inside_vehicle: bool                # Detected through vehicle window
    at_portal: Optional[str]            # Started/ended at portal
    property_entry: bool                # Came from outside property
    still_visible_at_end: bool          # Other agents present when ended
    exit_reason: str                    # "left_frame" | "chunk_boundary" | "portal_crossed"
```

**Key insight:** A LocalTrack is a video clip with metadata. All system logic operates on LocalTracks.

### Track Duration and Termination

**Tracks are variable length (not always 12s):**

- **Chunk duration:** 12 seconds (fixed)
- **Track duration:** Variable (agent may leave before chunk ends)

**Examples:**
```python
# Track spans entire chunk
track.start_time = T+0
track.end_time = T+12
track.exit_reason = "chunk_boundary"
track.next_track = (chunk_N+1, local_id_X)  # Continues in next chunk

# Track terminates mid-chunk (agent left frame)
track.start_time = T+3
track.end_time = T+7  # Only 4 seconds
track.exit_reason = "left_frame"
track.next_track = None  # No continuation

# Track starts mid-chunk (agent entered frame)
track.start_time = T+5
track.end_time = T+12
track.exit_reason = "chunk_boundary"
track.previous_track = None  # First appearance
track.next_track = (chunk_N+1, local_id_Y)  # Continues
```

**Why this matters:**
- Merging uses exit_reason to determine if agent actually left camera
- Tracks with next_track=None indicate agent is no longer visible on this camera
- Variable length is normal - agent movement determines duration, not chunk boundaries

---

## The Four Handlers

### Handler 1: ChunkProcessor
**Purpose:** Convert video into LocalTracks

**Input:** 12-second video chunk from one camera
**Output:** List of LocalTracks (one per YOLO tracking ID)
**Timing:** Every 10 seconds (overlapping chunks)

**What it does:**
1. Run YOLO11 tracking on video chunk
2. Group detections by YOLO tracking ID
3. Create LocalTrack for each ID
4. Extract crops (body + face)
5. Compute quality scores
6. Save frames to disk

**What it does NOT do:**
- Link to previous chunks (TemporalLinker does this)
- Assign global_agent_id (IdentityResolver does this)
- Merge across cameras (IdentityResolver does this)

---

### Handler 2: TemporalLinker
**Purpose:** Link LocalTracks across chunk boundaries (same camera)

**Input:**
- New chunk's LocalTracks
- Previous chunk's LocalTracks (from 2s overlap zone)

**Output:** Updated previous_track/next_track pointers

**Timing:** Immediately after ChunkProcessor completes new chunk

**What it does:**
1. Find LocalTracks in overlap zone (last 2s of chunk N, first 2s of chunk N+1)
2. Match tracks using spatial continuity + appearance similarity
3. Set previous/next pointers (creates linked list)
4. Propagate global_agent_id if already resolved
5. Propagate identity_candidates if uncertain
6. Detect concurrent agents (multiple tracks at same time)

**What it does NOT do:**
- Cross-camera merging (IdentityResolver does this)
- Create new global agents (IdentityResolver does this)

**Example:**
```python
# Chunk N: YOLO_ID=3 (Gardener)
# Chunk N+1: YOLO_ID=5 (same person, YOLO reassigned ID)

TemporalLinker matches them:
  chunk_N.local_tracks[3].next_track = (chunk_N+1.id, 5)
  chunk_N+1.local_tracks[5].previous_track = (chunk_N.id, 3)

  # Propagate identity
  chunk_N+1.local_tracks[5].global_agent_id = chunk_N.local_tracks[3].global_agent_id
```

---

### Handler 3: IdentityResolver
**Purpose:** Assign global_agent_id to LocalTracks (cross-camera)

**Input:** LocalTrack without global_agent_id
**Output:** global_agent_id OR identity_candidates

**Timing:** After TemporalLinker (can be batched, runs periodically)

**What it does:**
1. Check if identity inherited from previous_track (if yes, done)
2. Find cross-camera merge candidates (using 6x6 grid, timing, exit cells)
3. Apply evidence hierarchy:
   - Face match → definitive
   - Grid overlap + single agent → definitive
   - Grid transition + alibi check → definitive or ambiguous
   - No evidence → new agent or ambiguous
4. Assign global_agent_id OR identity_candidates
5. Update merged_tracks relationships

**Evidence hierarchy:**
```
1. Face recognition (confidence ≥ 0.75) → AUTO-MERGE
2. Grid overlap (<0.4s travel) + single agent → AUTO-MERGE
3. Grid transition (fits learned path) + alibi passes → AUTO-MERGE
4. Grid transition + alibi fails → AMBIGUOUS (identity_candidates)
5. No grid match → NEW AGENT or QUEUE FOR REVIEW
```

**What it does NOT do:**
- Process evidence that arrives later (EvidenceProcessor does this)
- Force resolution when uncertain (leaves identity_candidates set)

**Example:**
```python
# LocalTrack on Camera B (no global_agent_id yet)
# Exit from Camera A matches in grid

candidates = find_cross_camera_candidates(track_B)
# Returns: [Agent_A] (single candidate)

IdentityResolver:
  track_B.global_agent_id = Agent_A
  track_B.merged_tracks.append((camera_A, chunk_X, local_id_Y))
```

---

### Handler 4: EvidenceProcessor
**Purpose:** Resolve uncertain tracks when evidence arrives

**Input:**
- Evidence (face match result, LLM decision, human review)
- LocalTracks with identity_candidates

**Output:** Updated global_agent_id on affected LocalTracks

**Timing:** Async (triggered when evidence arrives)

**What it does:**
1. Find LocalTracks affected by evidence
2. Filter identity_candidates based on evidence
3. If single candidate remains → resolve to global_agent_id
4. Propagate through relationships (previous/next/merged)
5. Apply process of elimination (if Agent_A confirmed elsewhere, remove from other candidates)

**Example:**
```python
# Two people exit garage, both uncertain:
track_1.identity_candidates = {Agent_A, Agent_C}
track_2.identity_candidates = {Agent_A, Agent_C}

# Face match arrives: track_1 = Agent_A
EvidenceProcessor:
  track_1.global_agent_id = Agent_A
  track_1.identity_candidates.clear()

  # Eliminate A from track_2
  track_2.identity_candidates.remove(Agent_A)
  # Only Agent_C remains
  track_2.global_agent_id = Agent_C
  track_2.identity_candidates.clear()
```

---

## Data Flow

```
Video Chunks (every 10s)
    ↓
[ChunkProcessor]
    ↓
LocalTracks (unlinked, no global_agent_id)
    ↓
[TemporalLinker]
    ↓
LocalTracks (linked same-camera, may have global_agent_id from previous)
    ↓
[IdentityResolver] (periodic, e.g., every 30s)
    ↓
LocalTracks (with global_agent_id OR identity_candidates)
    ↓
[EvidenceProcessor] (async, when evidence arrives)
    ↓
LocalTracks (uncertainty resolved)
```

**End state:** Every LocalTrack has global_agent_id (certain) OR identity_candidates (uncertain).

---

## Timing & Triggers

### Real-time Processing (every 10s)
```
T=0:   Process chunk 0 (T=0-12s) on all cameras
T=10:  Process chunk 1 (T=10-22s) on all cameras, link to chunk 0
T=20:  Process chunk 2 (T=20-32s) on all cameras, link to chunk 1
...
```

**Per chunk:**
1. ChunkProcessor creates LocalTracks (~50ms per camera)
2. TemporalLinker links to previous chunk (~10ms per camera)
3. LocalTracks saved to database

### Batch Processing (every 30s)
```
T=30:  IdentityResolver processes all unresolved LocalTracks from T=0-30s
T=60:  IdentityResolver processes all unresolved LocalTracks from T=30-60s
...
```

**Why batched:** Cross-camera merging needs to see recent tracks from all cameras. Batching is more efficient than per-chunk.

### Event-driven Processing (async)
```
Face match completes → EvidenceProcessor updates affected LocalTracks
LLM returns decision → EvidenceProcessor updates affected LocalTracks
Human reviews → EvidenceProcessor updates affected LocalTracks
```

---

## GlobalAgent: Not Stored, Just Queried

**GlobalAgent is not a database table. It's a query result.**

```python
def get_agent_timeline(global_agent_id: int) -> List[LocalTrack]:
    """Get all tracks for this agent, ordered by time"""
    return db.query(LocalTrack).filter(
        global_agent_id == global_agent_id
    ).order_by(start_time).all()

def get_agent_current_state(global_agent_id: int) -> str:
    """Where is this agent now?"""
    latest = db.query(LocalTrack).filter(
        global_agent_id == global_agent_id
    ).order_by(start_time.desc()).first()

    if not latest:
        return "unknown"

    if latest.inside_vehicle:
        return "in_vehicle"
    elif latest.at_portal:
        return f"in_structure({latest.at_portal})"
    elif latest.exit_reason == "property_exit":
        return "left_property"
    else:
        return f"visible({latest.camera_id})"
```

**Why no GlobalAgent table:**
- All state is derived from LocalTracks
- No synchronization issues
- Simpler schema
- Easier to debug (just query LocalTracks)

---

## Handling Uncertainty

**Uncertainty is just identity_candidates on LocalTracks.**

### Example 1: Cross-camera ambiguity
```python
# Person exits Camera A, two candidates on Camera B
track_B1.identity_candidates = {Agent_A, Agent_B}
track_B2.identity_candidates = {Agent_A, Agent_B}

# Face match resolves track_B1 = Agent_A
# EvidenceProcessor eliminates Agent_A from track_B2
track_B2.global_agent_id = Agent_B  # Resolved by elimination
```

### Example 2: Structure exit ambiguity
```python
# 4 people enter garage
garage_entry_tracks = [track_A, track_B, track_C, track_D]
# All have global_agent_id (certain)

# 2 people exit garage
garage_exit_tracks = [track_X, track_Y]
track_X.identity_candidates = {Agent_A, Agent_B, Agent_C, Agent_D}
track_Y.identity_candidates = {Agent_A, Agent_B, Agent_C, Agent_D}

# Re-ID matches track_X to Agent_A, track_Y to Agent_C
# EvidenceProcessor resolves both
```

### Example 3: Vehicle occupancy uncertainty
```python
# Vehicle exits garage (4 people inside)
vehicle_track.property_entry = True  # Came from structure
vehicle_track.context_note = "exited_garage_with_occupants"

# 2 people later emerge from garage
# Those 2 are NOT in vehicle (process of elimination)
# But which 2 are IN vehicle? Unknown until vehicle parks and people exit
```

**Key principle:** Don't force resolution. Leave identity_candidates set until evidence resolves it.

---

## Special Cases (Just Context Flags)

### Concurrent Agents
**What:** Multiple people in same camera at same time
**How:** LocalTrack.concurrent_tracks list
**Impact:** YOLO keeps them separate (different local_ids), so no special handling needed

### Vehicle Occupancy
**What:** Person detected through vehicle window
**How:** LocalTrack.inside_vehicle = True
**Impact:** Sporadic detections, excluded from grid learning, used for face matching only

### Structure Entry/Exit
**What:** Person crosses portal (enters building)
**How:** LocalTrack.at_portal = "garage_door", exit_reason = "portal_crossed"
**Impact:** Creates identity_candidates for next person exiting same portal

### Property Entry
**What:** Vehicle/person arrives from outside property
**How:** LocalTrack.property_entry = True
**Impact:** Unknown initial state (vehicle could have hidden occupants)

**All of these are just flags on LocalTracks. No separate handling logic.**

---

## What Gets Built When (Phases)

### Phase 0: Foundation (Weeks 1-2)
- ChunkProcessor (YOLO → LocalTracks)
- Frame storage
- Database schema (LocalTrack table)

**Success:** Can create and store LocalTracks

---

### Phase 1: Single-Camera Tracking (Weeks 3-4)
- TemporalLinker (link same-camera chunks)
- LocalTrack.previous_track/next_track

**Success:** Can follow one person across chunks on one camera

---

### Phase 2: Cross-Camera Identity (Weeks 5-8)
- IdentityResolver (cross-camera merging)
- 6x6 grid learning
- Face recognition integration
- EvidenceProcessor (human/LLM review)

**Success:** Can merge person across multiple cameras

---

### Phase 3+: Refinement (Months 3+)
- Improve grid learning
- Add portal configuration
- Vehicle/structure handling
- Process of elimination
- Timeline viewer

**Success:** 90%+ auto-merge rate, minimal human review

---

## Database Schema (Simplified)

```sql
-- Only one table needed
CREATE TABLE local_tracks (
    id SERIAL PRIMARY KEY,

    -- Identity
    local_id INT NOT NULL,
    camera_id VARCHAR NOT NULL,
    chunk_id INT NOT NULL,

    -- Timing
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,

    -- Spatial
    entry_cell_x INT,
    entry_cell_y INT,
    exit_cell_x INT,
    exit_cell_y INT,

    -- Relationships (stored as references)
    previous_track_id INT REFERENCES local_tracks(id),
    next_track_id INT REFERENCES local_tracks(id),

    -- Identity (resolved or uncertain)
    global_agent_id INT,  -- NULL if uncertain
    identity_candidates INT[],  -- Array of possible agent IDs

    -- Context flags
    inside_vehicle BOOLEAN DEFAULT FALSE,
    at_portal VARCHAR,
    property_entry BOOLEAN DEFAULT FALSE,
    still_visible_at_end BOOLEAN DEFAULT FALSE,
    exit_reason VARCHAR,

    -- Detections (JSON for flexibility)
    detections JSONB,

    INDEX (global_agent_id),
    INDEX (camera_id, chunk_id),
    INDEX (start_time)
);

-- That's it. No GlobalAgent table, no Constraint table, no Container table.
```

---

## Queries: The Real "GlobalAgent"

```sql
-- Get agent timeline
SELECT * FROM local_tracks
WHERE global_agent_id = 123
ORDER BY start_time;

-- Get current agent state
SELECT * FROM local_tracks
WHERE global_agent_id = 123
ORDER BY start_time DESC
LIMIT 1;

-- Find uncertain tracks (need review)
SELECT * FROM local_tracks
WHERE global_agent_id IS NULL
  AND array_length(identity_candidates, 1) > 0;

-- Who's in garage?
SELECT DISTINCT global_agent_id
FROM local_tracks
WHERE at_portal = 'garage_door'
  AND exit_reason = 'portal_crossed'
  AND global_agent_id NOT IN (
    SELECT global_agent_id
    FROM local_tracks
    WHERE at_portal = 'garage_door'
      AND start_time > subquery.end_time
  );
```

**The database IS the timeline. The queries ARE the agent state.**

---

## Related Documents

### Strategy (L2)
- **L2_Strategy.md** - Overview and principles
- **L2_Strategy_Principles.md** - Core principles (evidence-based, no probabilities)

### Tactics (L3)
- **L3_ChunkProcessing.md** - ChunkProcessor implementation tactics
- **L3_TemporalLinking.md** - TemporalLinker implementation tactics
- **L3_IdentityResolution.md** - IdentityResolver implementation tactics
- **L3_EvidenceProcessing.md** - EvidenceProcessor implementation tactics

### Technical (L13)
- **L13_LocalTrack.md** - LocalTrack data structure and schema
- **L13_Handlers.md** - Handler implementations (ChunkProcessor, TemporalLinker, etc.)
- **L13_Queries.md** - Timeline queries, playback, state reconstruction

### Reference (L10)
- **L10_Nomenclature.md** - Terminology (LocalTrack, handlers, relationships)
