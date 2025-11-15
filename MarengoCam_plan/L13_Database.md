# Level 13 - Database Implementation Spec

## Purpose
Database schema for MarengoCam. Implementation-ready SQL specification.

---

## Schema

### local_tracks Table

**Purpose:** Store all LocalTrack observations. This is the only table needed.

```sql
CREATE TABLE local_tracks (
    -- Primary key
    id SERIAL PRIMARY KEY,

    -- Identity within chunk
    local_id INT NOT NULL,
    camera_id VARCHAR NOT NULL,
    chunk_id INT NOT NULL,

    -- Timing
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,

    -- Spatial (6x6 grid)
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
    exit_reason VARCHAR,  -- "left_frame" | "chunk_boundary" | "portal_crossed"

    -- Detections (frame-by-frame bboxes)
    detections JSONB
);
```

---

## Indexes

```sql
-- Query by agent (most common)
CREATE INDEX idx_global_agent_id ON local_tracks(global_agent_id);

-- Query by camera/chunk (chunk processing)
CREATE INDEX idx_camera_chunk ON local_tracks(camera_id, chunk_id);

-- Query by time (timeline reconstruction)
CREATE INDEX idx_start_time ON local_tracks(start_time);

-- Query uncertain tracks (review queue)
CREATE INDEX idx_identity_candidates ON local_tracks USING GIN(identity_candidates);
```

---

## Constraints

```sql
-- Grid cells must be valid (0-5 for 6x6 grid)
ALTER TABLE local_tracks ADD CONSTRAINT check_entry_cell_x CHECK (entry_cell_x BETWEEN 0 AND 5);
ALTER TABLE local_tracks ADD CONSTRAINT check_entry_cell_y CHECK (entry_cell_y BETWEEN 0 AND 5);
ALTER TABLE local_tracks ADD CONSTRAINT check_exit_cell_x CHECK (exit_cell_x BETWEEN 0 AND 5);
ALTER TABLE local_tracks ADD CONSTRAINT check_exit_cell_y CHECK (exit_cell_y BETWEEN 0 AND 5);

-- Exit reason must be valid enum
ALTER TABLE local_tracks ADD CONSTRAINT check_exit_reason
    CHECK (exit_reason IN ('left_frame', 'chunk_boundary', 'portal_crossed'));

-- Start time must be before end time
ALTER TABLE local_tracks ADD CONSTRAINT check_time_order
    CHECK (start_time < end_time);
```

---

## Detections JSON Schema

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "frame_time": {"type": "number"},
      "bbox": {
        "type": "object",
        "properties": {
          "x": {"type": "integer"},
          "y": {"type": "integer"},
          "width": {"type": "integer"},
          "height": {"type": "integer"}
        }
      },
      "confidence": {"type": "number"},
      "class": {"type": "string", "enum": ["person", "vehicle", "animal"]}
    }
  }
}
```

---

## Common Queries

### Get agent timeline

```sql
SELECT * FROM local_tracks
WHERE global_agent_id = $1
ORDER BY start_time;
```

### Get uncertain tracks

```sql
SELECT * FROM local_tracks
WHERE global_agent_id IS NULL
  AND cardinality(identity_candidates) > 0
ORDER BY start_time DESC;
```

### Get tracks in overlap zone (for TemporalLinker)

```sql
SELECT * FROM local_tracks
WHERE camera_id = $1
  AND chunk_id = $2
  AND end_time >= $3  -- overlap_start_time
ORDER BY start_time;
```

### Get tracks at timestamp (for alibi check)

```sql
SELECT * FROM local_tracks
WHERE start_time <= $1
  AND end_time >= $1
  AND global_agent_id IS NOT NULL;
```

---

## Migration Strategy

**Initial schema:** Create table with all fields

**Future migrations:** Add columns for new features (e.g., `movement_score`, `quality_score`)

**Data retention:** Archive old tracks (> 90 days) to cold storage, keep indexes on recent data

---

---

## Grid Learning Table

**Purpose:** Store learned travel times between camera pairs (6×6×6×6 matrix per pair per agent class).

```sql
CREATE TABLE grid_learning (
    -- Camera pair
    from_camera VARCHAR NOT NULL,
    to_camera VARCHAR NOT NULL,
    agent_class VARCHAR NOT NULL,  -- 'person' | 'vehicle'

    -- 6×6×6×6 matrix stored as JSONB
    -- Access: matrix[from_cell_x][from_cell_y][to_cell_x][to_cell_y] = travel_time_seconds
    -- Unlearned paths: 1000.0 (sentinel value)
    -- Learned paths: minimum observed travel time (float)
    matrix JSONB NOT NULL,

    -- Cached minimum across entire matrix (updated on each write)
    fastest_time FLOAT NOT NULL,

    -- Tracking
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (from_camera, to_camera, agent_class)
);

CREATE INDEX idx_grid_camera_pair ON grid_learning(from_camera, to_camera);
```

**Initialization:**
- Pre-populate all camera pairs with 1000.0 (unlearned)
- When learning: update cell if `new_time < matrix[from_x][from_y][to_x][to_y]`
- Recalculate `fastest_time` on each matrix update

**Adding cameras:**
- New camera creates 2×(N-1) new directed pairs × 2 agent classes
- Initialize new rows with matrix of 1000.0

---

## Face Crops Table

**Purpose:** Store face crop references for face recognition library.

```sql
CREATE TABLE face_crops (
    id SERIAL PRIMARY KEY,
    track_id INT REFERENCES local_tracks(id),
    global_agent_id INT,  -- NULL if uncertain during collection

    -- File storage
    file_path VARCHAR NOT NULL,  -- e.g., "/data/faces/agent_123/cam_A_12345.jpg"

    -- Metadata
    timestamp TIMESTAMP NOT NULL,
    camera_id VARCHAR NOT NULL,

    -- Face library management
    validated BOOLEAN DEFAULT FALSE,  -- Human validated for training
    in_library BOOLEAN DEFAULT FALSE  -- Currently in CodeProject.AI library
);

CREATE INDEX idx_face_agent ON face_crops(global_agent_id);
CREATE INDEX idx_face_validated ON face_crops(validated) WHERE validated = TRUE;
```

**Note:** Face crop selection for library is human decision during bootstrap, not automatic scoring

---

## Merged Tracks Relationship

**NEEDS SPEC:** Decide between:

**Option A: Junction table**
```sql
CREATE TABLE track_merges (
    track_a_id INT REFERENCES local_tracks(id),
    track_b_id INT REFERENCES local_tracks(id),
    merge_confidence VARCHAR,  -- 'face_match' | 'grid_overlap' | 'alibi' | 'human'
    merged_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (track_a_id, track_b_id)
);
```

**Option B: Array in local_tracks**
```sql
ALTER TABLE local_tracks ADD COLUMN merged_track_ids INT[];
```

**Decision needed:** Which approach for cross-camera track relationships?

---

## Camera State Table

**Purpose:** Track camera state for GPU allocation and chunk processing triggers.

```sql
CREATE TABLE camera_state (
    camera_id VARCHAR PRIMARY KEY,
    current_state VARCHAR NOT NULL,  -- 'Standby' | 'Armed' | 'Active' | 'Post'
    state_entered_at TIMESTAMP NOT NULL,
    last_chunk_end_time TIMESTAMP,  -- When last chunk was processed (for overlap calculation)
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_camera_active ON camera_state(current_state) WHERE current_state IN ('Armed', 'Active', 'Post');
```

**Usage:**
- GPU allocation: Query cameras in Active state (need full YOLO inference)
- Chunk processing: Track last_chunk_end_time to compute 12s chunk boundaries
- State transitions managed by ChunkProcessor

**Persistence:** Survives restarts (camera state not recomputed)

---

## Review Queue

**NEEDS SPEC:** Separate table or just query `local_tracks`?

**Option A: Query-based (no table)**
- Query uncertain tracks: `WHERE global_agent_id IS NULL AND cardinality(identity_candidates) > 0`
- Calculate priority on-the-fly

**Option B: Separate queue table**
```sql
CREATE TABLE review_queue (
    track_id INT REFERENCES local_tracks(id) PRIMARY KEY,
    priority_score FLOAT,
    reason VARCHAR,  -- 'ambiguous_timing' | 'multiple_candidates'
    queued_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Decision needed:** Does review queue need persistence or compute on demand?

---

## Notes

- **Primary table:** `local_tracks` stores all observations. GlobalAgent is query result, not stored.
- **JSONB for detections:** Flexible schema for frame-by-frame data, indexed via GIN.
- **Array for identity_candidates:** PostgreSQL native array support, indexed via GIN.
- **Timestamps:** Use TIMESTAMP for precision, not DATE.
- **Grid learning:** Pre-populated with 1000.0, updated with minimum observed times.
- **Face crops:** File path references only, no automatic quality scoring.
- **Camera state:** Persisted for GPU allocation and restart recovery.
