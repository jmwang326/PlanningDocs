# Level 13 - Database Schema

## Purpose
This document provides the complete technical specification for the system's metadata database. It contains the definitive `CREATE TABLE` statements for all tables, including `agents`, `tracks`, `merges`, and `detections`.

The schema defined herein is recovered from the `deprecated/DATA_AND_STORAGE_ARCHITECTURE.md` document and serves as the single source of truth for all data persistence logic.

## Main Database: `metadata/marengo.db` (SQLite)

### `identities` Table (from Face Library)
While managed by the `face_library` component, this table is crucial as it provides the master list of named entities. The `agents` table will link to this.

```sql
-- Master identity registry (shared by face library and Re-ID)
CREATE TABLE identities (
    identity_id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,             -- "John Doe", "Honda Accord Blue", "Unknown_042"
    type TEXT NOT NULL,                    -- "person" or "vehicle"
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen TIMESTAMP,
    notes TEXT,                            -- "Delivery driver", "Neighbor's car"
    
    -- Statistics
    total_tracks INTEGER DEFAULT 0,        -- How many times tracked
    total_merges INTEGER DEFAULT 0,        -- How many validated merges
    first_appearance TIMESTAMP,
    
    -- Embedding counts
    face_count INTEGER DEFAULT 0,          -- How many face embeddings (persons only)
    reid_count INTEGER DEFAULT 0           -- How many Re-ID fingerprints (Phase 4+)
);

CREATE INDEX idx_identities_name ON identities(name);
CREATE INDEX idx_identities_type ON identities(type);
CREATE INDEX idx_identities_last_seen ON identities(last_seen DESC);
```

### `face_embeddings` Table
Stores the actual face embeddings, linked to an identity.

```sql
-- Face embeddings reference identities
CREATE TABLE face_embeddings (
    embedding_id INTEGER PRIMARY KEY AUTOINCREMENT,
    identity_id INTEGER NOT NULL,         -- FK to identities table
    embedding BLOB NOT NULL,              -- 512-dim float32 vector (2048 bytes)
    quality REAL NOT NULL,                -- Quality score 0.0-1.0
    source_track_id TEXT,                 -- Track that generated this
    source_camera TEXT,
    timestamp TIMESTAMP,
    image_path TEXT,                      -- Path to face crop
    FOREIGN KEY (identity_id) REFERENCES identities(identity_id)
);

CREATE INDEX idx_face_identity ON face_embeddings(identity_id);
CREATE INDEX idx_face_quality ON face_embeddings(quality DESC);
```

### `agents` Table
Represents persistent entities across cameras and time.

```sql
-- Agents (persistent entities across cameras/time)
CREATE TABLE agents (
    agent_id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_class TEXT NOT NULL,             -- 'person', 'vehicle', 'animal'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen TIMESTAMP,
    identity_id INTEGER,                   -- Foreign key to identities table (face_library) if named
    status TEXT DEFAULT 'active',          -- 'active', 'archived', 'merged'
    merged_into_agent_id INTEGER,           -- If merged, points to canonical agent
    FOREIGN KEY (identity_id) REFERENCES identities(identity_id) ON DELETE SET NULL
);
CREATE INDEX idx_agents_status ON agents(status);
CREATE INDEX idx_agents_identity ON agents(identity_id);
```

### `tracks` Table
Represents observable windows of an agent on a single camera. An agent can have many tracks.

```sql
-- Track segments (observable windows on ONE camera)
CREATE TABLE tracks (
    track_id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id INTEGER NOT NULL,             -- Which agent this track belongs to
    camera TEXT NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    first_cell_row INTEGER,               -- 32×32 fine grid
    first_cell_col INTEGER,
    last_cell_row INTEGER,
    last_cell_col INTEGER,
    first_cell_8x8_row INTEGER,           -- 8×8 coarse grid (portal learning)
    first_cell_8x8_col INTEGER,
    last_cell_8x8_row INTEGER,
    last_cell_8x8_col INTEGER,
    detection_count INTEGER DEFAULT 0,
    max_confidence REAL,
    
    -- Multi-person tracking metadata
    solo_track BOOLEAN DEFAULT 1,         -- Only one agent in all frames (for clean learning)
    max_concurrent_agents INTEGER DEFAULT 1,  -- Max agents detected together in any frame
    
    -- ID-swap detection
    possibly_switched BOOLEAN DEFAULT 0,   -- ID swap flag (track may have wrong agent_id)
    
    -- Unbalanced group handling (Phase 3+)
    exit_type TEXT,                       -- 'portal', 'exit_boundary', 'occlusion', 'camera_edge', 'timeout', 'stayed_behind'
    exit_portal_id TEXT,                  -- Which portal if exit_type='portal'
    group_id INTEGER,                     -- Links tracks detected together (multi-person groups)
    candidate_agents TEXT,                -- JSON array: "[7,8,9]" for ambiguous assignment
    
    status TEXT DEFAULT 'active',          -- 'active', 'ended', 'merged'
    FOREIGN KEY (agent_id) REFERENCES agents(agent_id) ON DELETE CASCADE
);
CREATE INDEX idx_tracks_camera ON tracks(camera);
CREATE INDEX idx_tracks_time ON tracks(start_time, end_time);
CREATE INDEX idx_tracks_agent ON tracks(agent_id);
CREATE INDEX idx_tracks_solo ON tracks(solo_track);
CREATE INDEX idx_tracks_group ON tracks(group_id);
```

### `detections` Table
Stores sampled detection data for each track. Keyframes are prioritized to keep the table size manageable.

```sql
-- Detection logs (sampled, not every frame)
CREATE TABLE detections (
    detection_id INTEGER PRIMARY KEY AUTOINCREMENT,
    track_id INTEGER NOT NULL,
    camera TEXT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    bbox_x1 INTEGER, bbox_y1 INTEGER, bbox_x2 INTEGER, bbox_y2 INTEGER,
    cell_row INTEGER, cell_col INTEGER,
    confidence REAL,
    class TEXT,
    frame_path TEXT,                       -- Path to JPEG
    is_keyframe BOOLEAN DEFAULT 0,         -- Keyframe sampling for long tracks
    FOREIGN KEY (track_id) REFERENCES tracks(track_id) ON DELETE CASCADE
);
CREATE INDEX idx_detections_track ON detections(track_id);
CREATE INDEX idx_detections_time ON detections(timestamp);
CREATE INDEX idx_detections_keyframe ON detections(is_keyframe);
```

### `merges` Table
Logs all merge decisions, the evidence used, and the final validation outcome. This is the audit trail for the system's reasoning.

```sql
-- Merge decisions
CREATE TABLE merges (
    merge_id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_track_id INTEGER NOT NULL,
    dest_track_id INTEGER NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    delta_t REAL,                          -- Time gap in seconds (negative if concurrent)
    portal_id TEXT,                        -- Portal used (if any)
    decision TEXT NOT NULL,                -- AUTO_MERGE, REVIEW_LLM, REVIEW_HUMAN, REJECT
    strong_evidence_count INTEGER,
    weak_evidence_count INTEGER,
    
    -- Similarity scores
    face_similarity REAL,                  -- Face match confidence (0.0-1.0)
    reid_similarity REAL,                  -- Re-ID match confidence (0.0-1.0)
    reid_second_best REAL,                 -- Second-best Re-ID candidate similarity
    reid_margin REAL,                      -- best - second_best (disambiguation confidence)
    
    -- Spatial metrics
    spatial_distance_m REAL,               -- Euclidean distance between track endpoints (meters)
    expected_distance_m REAL,              -- Based on walking_speed × delta_t
    plausibility_ratio REAL,               -- actual / expected (>2.5 = implausible)
    
    -- Multi-person metadata
    possible_switched_agent BOOLEAN DEFAULT 0,  -- Multi-person ambiguity flag
    source_concurrent_agents INTEGER,      -- Denormalized from tracks (for fast queries)
    dest_concurrent_agents INTEGER,
    group_merge_id INTEGER,                -- Links merges from same multi-person group
    
    -- Validation
    llm_confidence REAL,
    human_validated BOOLEAN DEFAULT 0,
    validation_outcome TEXT,               -- ACCEPT, REJECT, DEFER
    validation_timestamp TIMESTAMP,
    reviewed_by TEXT,                      -- 'human', 'llm', 'auto_face', 'auto_reid'
    correction_type TEXT,                  -- 'split', 'reassign', 'confirmed', NULL
    original_decision TEXT,                -- What system suggested before human override
    
    panel_path TEXT,                       -- Path to decision panels
    FOREIGN KEY (source_track_id) REFERENCES tracks(track_id) ON DELETE CASCADE,
    FOREIGN KEY (dest_track_id) REFERENCES tracks(track_id) ON DELETE CASCADE
);
CREATE INDEX idx_merges_decision ON merges(decision);
CREATE INDEX idx_merges_validated ON merges(human_validated);
CREATE INDEX idx_merges_switched ON merges(possible_switched_agent);
CREATE INDEX idx_merges_group ON merges(group_merge_id);
```

### `camera_states` Table
Logs camera state transitions for debugging and performance analysis.

```sql
-- Camera states (for debugging state transitions)
CREATE TABLE camera_states (
    state_id INTEGER PRIMARY KEY AUTOINCREMENT,
    camera TEXT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    old_state TEXT,
    new_state TEXT NOT NULL,               -- Standby, Armed, Active, Post
    trigger TEXT,                          -- What caused transition
    active_tracks INTEGER DEFAULT 0
);
CREATE INDEX idx_states_camera ON camera_states(camera);
CREATE INDEX idx_states_time ON camera_states(timestamp);
```

### `inter_camera_adjacency` Table
Stores the learned `6x6` grid data, representing the minimum travel time between cells of a camera pair.

```sql
-- Learned 6x6 grid data for inter-camera transitions
CREATE TABLE inter_camera_adjacency (
    source_camera_id TEXT NOT NULL,
    dest_camera_id TEXT NOT NULL,
    agent_class TEXT NOT NULL,
    source_cell_row INTEGER NOT NULL,
    source_cell_col INTEGER NOT NULL,
    dest_cell_row INTEGER NOT NULL,
    dest_cell_col INTEGER NOT NULL,
    min_travel_time_ms INTEGER NOT NULL,
    PRIMARY KEY (source_camera_id, dest_camera_id, agent_class, source_cell_row, source_cell_col, dest_cell_row, dest_cell_col)
);
```

### `events` Table
Groups multi-camera activity into coherent events.

```sql
-- Events (multi-camera activity groups)
CREATE TABLE events (
    event_id INTEGER PRIMARY KEY AUTOINCREMENT,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    cameras TEXT,                          -- JSON array of camera names
    agent_ids TEXT,                        -- JSON array of agent_ids involved
    manifest_path TEXT,                    -- Path to event manifest JSON
    clip_path TEXT,                        -- Path to reconstructed MP4 (optional)
    status TEXT DEFAULT 'active'           -- 'active', 'closed', 'archived'
);
CREATE INDEX idx_events_time ON events(start_time);
```

### `validations` Table
Stores ground truth from human or LLM reviews, used for learning and accuracy metrics.

```sql
-- Validations (ground truth for learning)
CREATE TABLE validations (
    validation_id INTEGER PRIMARY KEY AUTOINCREMENT,
    merge_id INTEGER NOT NULL,
    validated_by TEXT,                     -- 'llm' or 'user'
    outcome TEXT NOT NULL,                 -- ACCEPT, REJECT, DEFER
    confidence REAL,                       -- LLM confidence or 1.0 for human
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notes TEXT,
    FOREIGN KEY (merge_id) REFERENCES merges(merge_id) ON DELETE CASCADE
);
CREATE INDEX idx_validations_merge ON validations(merge_id);
```

### `schema_version` Table
Tracks the current version of the database schema to manage migrations.

```sql
-- Use Alembic or simple versioning table
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO schema_version (version) VALUES (1);
```
