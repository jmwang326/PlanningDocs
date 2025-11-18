# L3T: Database Schema

This document provides the definitive database schema for the MarengoCam system. The schema is designed around a single source of truth: the `local_tracks` table.

---

## Core Principle: The `local_tracks` Table

The entire system is built on the principle that `GlobalAgent`s and `Timeline`s are query-time constructs, not stored objects. All raw observations and resolved states are stored in a single, canonical `local_tracks` table.

### `local_tracks` Schema

```sql
CREATE TABLE local_tracks (
    -- Core Identity & Primary Key
    id SERIAL PRIMARY KEY,
    camera_id VARCHAR NOT NULL,
    chunk_id INT NOT NULL,
    local_id INT NOT NULL, -- The tracker's ID within the chunk

    -- Resolved Identity (The Goal)
    global_agent_id INT, -- NULL if uncertain

    -- Timing
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time TIMESTAMP WITH TIME ZONE NOT NULL,

    -- Spatial Information (6x6 Grid)
    entry_cell_x INT,
    entry_cell_y INT,
    exit_cell_x INT,
    exit_cell_y INT,

    -- Relationships
    previous_track_id INT REFERENCES local_tracks(id),
    next_track_id INT REFERENCES local_tracks(id),

    -- Uncertainty
    identity_candidates INT[], -- A postgres array of potential global_agent_ids

    -- Contextual Information
    exit_reason VARCHAR, -- "left_frame", "chunk_boundary", "portal_crossed"
    at_portal VARCHAR,   -- ID of the portal, if applicable

    -- Raw Data
    detections JSONB -- A JSON array of all frame-by-frame detections
);
```

### `local_tracks` Indexes

```sql
-- The most common query: retrieving a timeline for a specific agent.
CREATE INDEX idx_global_agent_id ON local_tracks(global_agent_id);

-- Used by the EvidenceProcessor to find all uncertain tracks.
CREATE INDEX idx_identity_candidates ON local_tracks USING GIN(identity_candidates);

-- Used for forensic analysis and timeline reconstruction.
CREATE INDEX idx_start_time ON local_tracks(start_time);

-- Used by the TemporalLinker to find tracks in the overlap zone.
CREATE INDEX idx_camera_chunk ON local_tracks(camera_id, chunk_id);
```

---

## Supporting Tables

### `grid_learning`

Stores the learned travel times between camera grid cells. This is the system's spatial memory.

```sql
CREATE TABLE grid_learning (
    from_camera VARCHAR NOT NULL,
    to_camera VARCHAR NOT NULL,
    agent_class VARCHAR NOT NULL, -- 'person' or 'vehicle'
    matrix JSONB NOT NULL, -- A multi-dimensional array storing travel times
    PRIMARY KEY (from_camera, to_camera, agent_class)
);
```

### `track_merges`

Provides a permanent audit log of all merge events, enabling traceability and the "unmerging" of incorrect decisions.

```sql
CREATE TABLE track_merges (
    id SERIAL PRIMARY KEY,
    merge_timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    retained_agent_id INT NOT NULL, -- The agent that was kept
    consumed_agent_id INT NOT NULL, -- The agent that was merged and retired
    reason VARCHAR, -- e.g., 'face_match', 'human_review'
    is_reverted BOOLEAN DEFAULT FALSE -- If a human later split this merge
);
```

### `camera_state`

Persists the real-time operational state of each camera, allowing the system to recover its state after a restart.

```sql
CREATE TABLE camera_state (
    camera_id VARCHAR PRIMARY KEY,
    current_state VARCHAR NOT NULL, -- 'Standby', 'Armed', 'Active', 'Post'
    state_entered_at TIMESTAMP WITH TIME ZONE NOT NULL,
    last_chunk_end_time TIMESTAMP WITH TIME ZONE
);
```
