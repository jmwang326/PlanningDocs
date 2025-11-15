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

## Notes

- **One table only:** No GlobalAgent, no Constraint, no Container tables. All state derived from LocalTracks.
- **JSONB for detections:** Flexible schema for frame-by-frame data, indexed via GIN.
- **Array for identity_candidates:** PostgreSQL native array support, indexed via GIN.
- **Timestamps:** Use TIMESTAMP for precision, not DATE.
