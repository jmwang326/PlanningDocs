# Level 2 - Decision Architecture

## Purpose
Define the fundamental architecture of MarengoCam: what gets processed, who processes it, and when.

---

## Core Philosophy: Tracks + Handlers

**The entire system exists to manage video clips (tracks).**

Everything else - agents, timelines, uncertainty, vehicles, structures - is just metadata and queries on top of tracks.

---

## The Fundamental Unit: LocalTrack

```python
class LocalTrack:
    """
    One observation from YOLO tracking (up to 12 seconds).
    This is the atom of the system. Everything builds from this.
    """
    # Identity within chunk
    local_id: int              # YOLO's tracking ID (chunk-scoped)
    camera_id: str
    chunk_id: int

    # Observations (what we saw)
    detections: List[Detection]  # Frame-by-frame bboxes
    start_time: float
    end_time: float            # Variable duration (agent may leave early)
    entry_cell: Tuple[int, int]  # 6x6 grid
    exit_cell: Tuple[int, int]

    # Relationships to other tracks
    previous_track: Optional[Tuple[int, int]]  # Same camera, previous chunk
    next_track: Optional[Tuple[int, int]]      # Same camera, next chunk
    merged_tracks: List[Tuple[str, int, int]]  # Other cameras, same agent
    concurrent_tracks: List[int]               # Same camera, different agents

    # Identity (resolved or uncertain)
    global_agent_id: Optional[int]      # Which agent (if known)
    identity_candidates: Set[int]       # Could be any of these (if uncertain)

    # Context flags (special circumstances)
    inside_vehicle: bool
    at_portal: Optional[str]
    property_entry: bool
    exit_reason: str  # "left_frame" | "chunk_boundary" | "portal_crossed"
```

**Key insight:** A LocalTrack is a video clip with metadata. All system logic operates on LocalTracks.

---

## The Six Components

### 1. ChunkProcessor
**What:** Converts video chunks into LocalTracks
**Input:** 12-second video chunk from one camera
**Output:** List of LocalTracks (one per YOLO tracking ID)
**Timing:** Every 10 seconds (overlapping chunks)

**See:** L3_ChunkProcessing.md for tactics

---

### 2. TemporalLinker
**What:** Links LocalTracks across chunk boundaries (same camera)
**Input:** New chunk's LocalTracks + previous chunk's LocalTracks
**Output:** Updated previous_track/next_track pointers
**Timing:** Immediately after ChunkProcessor

**See:** L3_TemporalLinking.md for tactics

---

### 3. IdentityResolver
**What:** Assigns global_agent_id to LocalTracks (cross-camera merging)
**Input:** LocalTrack without global_agent_id
**Output:** global_agent_id OR identity_candidates
**Timing:** Periodic (batched, e.g., every 30s)

**Evidence hierarchy:**
1. Face recognition (≥ 0.75 confidence) → AUTO-MERGE
2. Grid overlap (< 0.4s travel) + single agent → AUTO-MERGE
3. Grid transition (fits learned path) + alibi passes → AUTO-MERGE
4. Grid transition + alibi fails → AMBIGUOUS
5. No grid match → NEW AGENT or QUEUE FOR REVIEW

**See:** L3_IdentityResolution.md for tactics (grid learning, alibi checks, process of elimination)

---

### 4. EvidenceProcessor
**What:** Resolves uncertain tracks when evidence arrives
**Input:** Evidence (face match, LLM decision, human review) + LocalTracks with identity_candidates
**Output:** Updated global_agent_id
**Timing:** Async (event-driven)

**See:** L3_EvidenceProcessing.md for tactics

---

### 5. GUI (Timeline Viewer)
**What:** Displays agent timelines, review queue, playback
**Input:** Queries on LocalTracks
**Output:** Visual timeline, review panels

**See:** L3_Gui.md for tactics

---

### 6. Configuration Tool
**What:** Portal configuration, thresholds, camera settings
**Input:** User-defined portals (doors, gates), thresholds
**Output:** YAML configuration

**See:** L3_Configuration.md for tactics

---

### 7. System Health Monitor
**What:** Tracks system performance, GPU utilization, processing lag
**Input:** Metrics from handlers (chunk processing time, merge queue depth, GPU usage)
**Output:** Health dashboard, alerts

**See:** L3_SystemHealth.md for tactics

---

## Data Flow

```
Video Chunks (every 10s)
    ↓
[ChunkProcessor] → LocalTracks (unlinked)
    ↓
[TemporalLinker] → LocalTracks (linked same-camera)
    ↓
[IdentityResolver] → LocalTracks (with global_agent_id OR identity_candidates)
    ↓
[EvidenceProcessor] → LocalTracks (uncertainty resolved)
    ↓
[GUI] → Timeline display, review queue
```

**End state:** Every LocalTrack has global_agent_id (certain) OR identity_candidates (uncertain).

---

## GlobalAgent: Not Stored, Just Queried

**GlobalAgent is not a database table. It's a query result.**

```python
def get_agent_timeline(global_agent_id: int) -> List[LocalTrack]:
    """Get all tracks for this agent, ordered by time"""
    return db.query(LocalTrack).filter(
        global_agent_id == global_agent_id
    ).order_by(start_time).all()
```

**Why no GlobalAgent table:**
- All state is derived from LocalTracks
- No synchronization issues
- Simpler schema

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

    -- Relationships
    previous_track_id INT REFERENCES local_tracks(id),
    next_track_id INT REFERENCES local_tracks(id),

    -- Identity (resolved or uncertain)
    global_agent_id INT,  -- NULL if uncertain
    identity_candidates INT[],

    -- Context flags
    inside_vehicle BOOLEAN DEFAULT FALSE,
    at_portal VARCHAR,
    property_entry BOOLEAN DEFAULT FALSE,
    exit_reason VARCHAR,

    -- Detections (JSON)
    detections JSONB,

    INDEX (global_agent_id),
    INDEX (camera_id, chunk_id),
    INDEX (start_time)
);
```

**That's it.** No GlobalAgent table, no Constraint table, no Container table.

---

## Related Documents

### Strategy (L2)
- **L2_Strategy.md** - Strategic principles and approach
- **L2_Strategy_Principles.md** - Core principles (evidence-based, no probabilities)

### Tactics (L3)
- **L3_ChunkProcessing.md** - How ChunkProcessor creates tracks
- **L3_TemporalLinking.md** - How TemporalLinker connects tracks across chunks
- **L3_IdentityResolution.md** - How IdentityResolver merges across cameras (grid learning, alibi checks, process of elimination)
- **L3_EvidenceProcessing.md** - How EvidenceProcessor resolves uncertainty
- **L3_Gui.md** - Timeline viewer, review queue
- **L3_Configuration.md** - Portal configuration, thresholds
- **L3_SystemHealth.md** - Performance monitoring

### Technical (L13)
- **L13_LocalTrack.md** - LocalTrack data structure details
- **L13_Handlers.md** - Handler implementations
- **L13_Queries.md** - Timeline queries, playback

### Reference (L10)
- **L10_Nomenclature.md** - Terminology (LocalTrack, handlers, relationships)
