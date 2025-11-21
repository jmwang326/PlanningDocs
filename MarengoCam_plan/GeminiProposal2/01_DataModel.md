# 01 - Data Model & Storage Specification

## Purpose
This document defines the "Source of Truth" for MarengoCam. It specifies the Database Schema (SQL), the File Storage structure, and the Data Lifecycle policies.

**Philosophy:**
- **Hybrid Storage:** Metadata in DB (Postgres), Media on Disk (Filesystem).
- **Vector-First:** Face and Re-ID embeddings are first-class citizens (pgvector).
- **Audit-Ready:** Every merge/split decision is logged with a reason.

---

## 1. Storage Architecture

### 1.1. Database (PostgreSQL)
Used for structured data, relationships, and vector search.
- **Extension:** `pgvector` (for Face/Re-ID embeddings)
- **Extension:** `PostGIS` (Optional, for advanced spatial queries, but likely overkill here)

### 1.2. Filesystem (Media)
Used for high-bandwidth video and image data.
**Directory Structure:**
```text
/marengo_data/
  ├── /clips/
  │     └── /{YYYY-MM-DD}/{camera_id}/{HH}/{chunk_id}.mp4
  ├── /crops/
  │     └── /{YYYY-MM-DD}/{camera_id}/{track_id}_{timestamp}.jpg
  └── /exports/
        └── /training_sets/{dataset_name}/...
```

---

## 2. Database Schema (SQL Definitions)

### 2.1. Core Entities

#### `cameras`
Syncs with `config.yaml` but allows DB-side constraints.
```sql
CREATE TABLE cameras (
    id VARCHAR(50) PRIMARY KEY, -- e.g., "front_door"
    name VARCHAR(100),
    zone_id VARCHAR(50),        -- e.g., "driveway"
    is_active BOOLEAN DEFAULT TRUE
);
```

#### `global_entities` (The "People")
Represents a unique physical identity (Person or Vehicle).
```sql
CREATE TABLE global_entities (
    id SERIAL PRIMARY KEY,
    label VARCHAR(100),         -- User-assigned name ("The Mailman")
    type VARCHAR(20),           -- "person", "vehicle"
    first_seen_at TIMESTAMP,
    last_seen_at TIMESTAMP,
    is_confirmed BOOLEAN DEFAULT FALSE, -- True if Human validated
    notes TEXT
);
```

#### `local_tracks` (The "Observations")
A continuous track from a single camera.
```sql
CREATE TABLE local_tracks (
    id VARCHAR(64) PRIMARY KEY, -- UUID or "cam_timestamp_objid"
    camera_id VARCHAR(50) REFERENCES cameras(id),
    global_entity_id INTEGER REFERENCES global_entities(id), -- NULL if unknown
    
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    
    object_type VARCHAR(20),    -- "person", "car"
    confidence FLOAT,           -- YOLO confidence
    
    -- Spatial Data
    entry_zone VARCHAR(20),     -- "left", "right", "grid_0_0"
    exit_zone VARCHAR(20),
    
    -- Media References
    thumbnail_path VARCHAR(255),
    video_chunk_id VARCHAR(64)
);
CREATE INDEX idx_tracks_time ON local_tracks(start_time);
CREATE INDEX idx_tracks_entity ON local_tracks(global_entity_id);
```

### 2.2. Evidence & Biometrics

#### `evidence` (The "Glue")
Specific frames or crops used to make decisions.
```sql
CREATE TABLE evidence (
    id SERIAL PRIMARY KEY,
    local_track_id VARCHAR(64) REFERENCES local_tracks(id),
    
    type VARCHAR(20),           -- "face", "body_reid", "plate"
    image_path VARCHAR(255),
    
    embedding VECTOR(512),      -- pgvector embedding
    quality_score FLOAT,        -- How clear is this face?
    
    is_training_sample BOOLEAN DEFAULT FALSE -- Flag for "Gold Standard"
);
```

### 2.3. The "Brain" (Grid Learning)

#### `grid_links` (Spatial Logic)
Stores the learned travel times between zones.
```sql
CREATE TABLE grid_links (
    from_camera VARCHAR(50),
    from_zone VARCHAR(20),
    to_camera VARCHAR(50),
    to_zone VARCHAR(20),
    
    sample_count INTEGER DEFAULT 0,
    avg_time_seconds FLOAT,
    std_dev_seconds FLOAT,
    
    last_updated TIMESTAMP,
    
    PRIMARY KEY (from_camera, from_zone, to_camera, to_zone)
);
```

### 2.4. Operations & Audit

#### `audit_logs` (The "Black Box")
Immutable history of system and user actions.
```sql
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    actor VARCHAR(50),          -- "system:identity_resolver" or "user:admin"
    action VARCHAR(50),         -- "MERGE", "SPLIT", "DELETE", "CONFIRM"
    
    target_type VARCHAR(20),    -- "global_entity", "local_track"
    target_id VARCHAR(64),
    
    details JSONB               -- {"reason": "Face match 0.98", "previous_id": 12}
);
```

#### `system_health` (Metrics)
Snapshots of system performance.
```sql
CREATE TABLE system_health (
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    component VARCHAR(50),      -- "chunk_processor"
    metric_name VARCHAR(50),    -- "fps", "queue_depth", "gpu_mem"
    metric_value FLOAT
);

### 2.5. Configuration

#### `system_settings` (Global Config)
Key-value store for runtime configuration (replacing most of config.yaml).
```sql
CREATE TABLE system_settings (
    key VARCHAR(50) PRIMARY KEY,  -- e.g., "retention_days", "ai_confidence_threshold"
    value JSONB,                  -- Flexible storage: 7, 0.75, or {"complex": "obj"}
    description TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. Data Lifecycle & Retention

### 3.1. Retention Policies
Enforced by `Component_SystemHealth` (The Janitor).

| Data Type | Retention Period | Storage Location | Action |
| :--- | :--- | :--- | :--- |
| **Raw Video** | 7 Days | Filesystem | Delete file |
| **Thumbnails** | 30 Days | Filesystem | Delete file |
| **Training Crops** | Indefinite | Filesystem | Keep (if `is_training_sample=TRUE`) |
| **LocalTracks** | 90 Days | Database | Archive to CSV or Delete |
| **GlobalEntities** | Indefinite | Database | Keep metadata |
| **Audit Logs** | 1 Year | Database | Delete old rows |

### 3.2. "Survival Mode" Triggers
If disk usage > 90%:
1.  Delete **Raw Video** older than 24 hours.
2.  Delete **Thumbnails** older than 7 days.
3.  Stop saving new **Thumbnails** for low-confidence tracks.

---

## 4. Data Flow (The "Middleman" Logic)

1.  **Ingest:** `ChunkProcessor` writes to `local_tracks` and `evidence`.
2.  **Process:** `IdentityResolver` reads `local_tracks`, queries `grid_links`, and updates `global_entity_id`.
3.  **Serve:** `API` reads `global_entities` joined with `local_tracks` to show the User the full story.
4.  **Learn:** User confirms a match via `API` -> `IdentityResolver` updates `grid_links` stats.
