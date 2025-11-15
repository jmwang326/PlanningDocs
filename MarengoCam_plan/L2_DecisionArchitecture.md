# Level 2 - Decision Architecture

## Purpose
Architecture: what gets processed, who processes it, when.

---

## Core Philosophy: Tracks + Handlers

**The entire system exists to manage video clips (tracks).**

Everything else (agents, timelines, uncertainty) is metadata and queries on top of tracks.

---

## The Fundamental Unit: LocalTrack

```python
class LocalTrack:
    """One observation from YOLO tracking (up to 12 seconds)."""
    # Identity within chunk
    local_id: int              # YOLO's tracking ID (chunk-scoped)
    camera_id: str
    chunk_id: int

    # Observations
    detections: List[Detection]
    start_time: float
    end_time: float
    entry_cell: Tuple[int, int]  # 6x6 grid
    exit_cell: Tuple[int, int]

    # Relationships
    previous_track: Optional[Tuple[int, int]]  # Same camera, previous chunk
    next_track: Optional[Tuple[int, int]]      # Same camera, next chunk
    merged_tracks: List[Tuple[str, int, int]]  # Other cameras, same agent

    # Identity (resolved or uncertain)
    global_agent_id: Optional[int]
    identity_candidates: Set[int]

    # Context flags
    inside_vehicle: bool
    at_portal: Optional[str]
    exit_reason: str  # "left_frame" | "chunk_boundary" | "portal_crossed"
```

---

## The Seven Components

### 1. ChunkProcessor
Converts 12s video chunks into LocalTracks (one per YOLO tracking ID). Runs every 10s (overlapping chunks).

### 2. TemporalLinker
Links LocalTracks across chunk boundaries (same camera). Updates previous_track/next_track pointers.

### 3. IdentityResolver
Assigns global_agent_id to LocalTracks (cross-camera merging). Batched every ~30s.

**Evidence hierarchy:**
1. Face recognition (≥ 0.75) → AUTO-MERGE
2. Grid overlap (< 0.4s) + single agent → AUTO-MERGE
3. Grid transition + alibi passes → AUTO-MERGE
4. Grid transition + alibi fails → AMBIGUOUS
5. No match → NEW AGENT or QUEUE

### 4. EvidenceProcessor
Resolves uncertain tracks when evidence arrives (face match, LLM decision, human review). Event-driven.

### 5. GUI
Timeline viewer, review queue, playback. Queries LocalTracks.

### 6. Configuration
Portal configuration, thresholds, camera settings. YAML.

### 7. SystemHealth
Performance monitoring, alerts. GPU utilization, processing lag, queue depths.

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

## GlobalAgent: Query Result, Not Stored

```python
def get_agent_timeline(global_agent_id: int) -> List[LocalTrack]:
    return db.query(LocalTrack).filter(
        global_agent_id == global_agent_id
    ).order_by(start_time).all()
```

**Why:** All state derived from LocalTracks. No synchronization issues.

---

## Database Schema

```sql
CREATE TABLE local_tracks (
    id SERIAL PRIMARY KEY,
    local_id INT NOT NULL,
    camera_id VARCHAR NOT NULL,
    chunk_id INT NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    entry_cell_x INT,
    entry_cell_y INT,
    exit_cell_x INT,
    exit_cell_y INT,
    previous_track_id INT REFERENCES local_tracks(id),
    next_track_id INT REFERENCES local_tracks(id),
    global_agent_id INT,  -- NULL if uncertain
    identity_candidates INT[],
    inside_vehicle BOOLEAN DEFAULT FALSE,
    at_portal VARCHAR,
    exit_reason VARCHAR,
    detections JSONB,
    INDEX (global_agent_id),
    INDEX (camera_id, chunk_id),
    INDEX (start_time)
);
```

**That's it.** One table. No GlobalAgent, no Constraint, no Container tables.
