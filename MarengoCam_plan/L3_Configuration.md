# Level 3 - Configuration

## Purpose
Tactical guide for the MarengoCam configuration system. Defines how cameras, portals, thresholds, and system parameters are managed, validated, and hot-reloaded.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for Configuration component overview.

---

## Configuration Structure

### Master Configuration File

**File:** `marengo.yaml` (YAML format)

**Top-level sections:**
```yaml
system:           # System-wide settings
cameras:          # Camera definitions
portals:          # Portal locations
thresholds:       # Detection and merging thresholds
alerts:           # Alert configuration
external_ai:      # External AI service settings
storage:          # File storage paths
database:         # Database connection
mode:             # Operational mode (online_learning / autonomous)
```

---

## Camera Configuration

### Camera Definition

```yaml
cameras:
  - id: "Driveway"
    name: "Driveway Camera"
    enabled: true

    # RTSP streams
    streams:
      main:
        url: !secret driveway_main_url
        resolution: "1920x1080"
        fps: 25
      sub:
        url: !secret driveway_sub_url
        resolution: "640x480"
        fps: 10

    # Grid overlay (6x6)
    grid:
      width: 6
      height: 6
      calibration: "auto"  # or "manual"

    # Camera-specific thresholds
    thresholds:
      detection_confidence: 0.5
      motion_threshold: 0.02
      min_track_duration: 3.0  # seconds

    # State management
    initial_state: "Standby"
    armed_timeout: 30  # seconds
    post_roll: 15  # seconds
```

**Required fields:**
- `id`: Unique camera identifier (string, no spaces)
- `streams.sub.url`: Sub profile RTSP URL (used for detection)

**Optional fields:**
- `name`: Human-readable name (defaults to `id`)
- `enabled`: Enable/disable camera (default: true)
- `streams.main.url`: Main profile URL (high-res, for storage)
- `thresholds.*`: Override system defaults

---

### Camera States

**Standby:**
- Motion gate enabled
- Sparse YOLO sampling
- No frame persistence

**Armed:**
- Moderate YOLO sampling
- Accumulate candidate tracks
- Begin frame persistence (sub profile)

**Active:**
- Full YOLO inference (all frames within GPU budget)
- Create LocalTracks
- Frame persistence (sub + main profiles)

**Post:**
- Moderate YOLO sampling (watching for re-entry)
- Frame persistence (sub profile only)
- Duration: `post_roll` seconds

**See:** L3_ChunkProcessing.md for state transitions

---

## Portal Configuration

### Portal Definition

**Purpose:** Mark locations where agents can become concealed (garage doors, building entrances)

```yaml
portals:
  - id: "garage_door"
    name: "Garage Door"
    camera_id: "Driveway"
    cells: [[3, 2], [3, 3], [4, 2], [4, 3]]  # Grid cells
    type: "structure_entry"  # or "vehicle_entry"
    bidirectional: true
```

**Fields:**
- `id`: Unique portal identifier
- `camera_id`: Which camera sees this portal
- `cells`: List of [x, y] grid cells marking portal location
- `type`: Portal type (affects merging logic)
  - `structure_entry`: Building entrance (person enters/exits)
  - `vehicle_entry`: Garage/vehicle entry (person enters vehicle)
- `bidirectional`: Can agent enter and exit through same portal (default: true)

### Portal Types

**Structure Entry:**
- Agent enters building, becomes concealed
- Time gap irrelevant (can be inside for any duration)
- Merging relies on Re-ID + secondary evidence

**Vehicle Entry:**
- Agent enters vehicle, becomes concealed
- Vehicle may move to different camera
- Requires vehicle occupancy tracking

**See:** L3_IdentityResolution.md for portal transition logic

---

## Threshold Configuration

### Detection Thresholds

```yaml
thresholds:
  detection:
    # YOLO confidence thresholds (per class)
    person: 0.5
    vehicle: 0.6
    animal: 0.4

    # Motion gate (Standby cameras only)
    motion_threshold: 0.02  # Pixel-diff threshold

    # Track quality filtering
    min_track_duration: 3.0  # seconds (sustained movement)
    min_movement_score: 0.5  # 0.0-1.0 (avoid stationary objects)
```

---

### Identity Resolution Thresholds

```yaml
thresholds:
  identity_resolution:
    # Face recognition (CodeProject.AI)
    face_match_threshold: 0.75  # Auto-merge if ≥ 0.75

    # Re-ID appearance similarity
    reid_similarity_threshold: 0.70  # Supplementary evidence

    # Grid-based merging
    overlap_zone_threshold: 0.4  # seconds (< 0.4s = overlap)
    grid_timing_variance: 0.3  # seconds (±30% tolerance)

    # Process of elimination
    alibi_check_enabled: true
    alibi_threshold: 2  # Max alternative sources before queuing
```

**Face match threshold:**
- ≥ 0.75: Auto-merge (definitive)
- 0.60-0.74: Queue for review (uncertain)
- < 0.60: Weak evidence (use secondary)

**Grid timing variance:**
- Tolerance for learned travel times
- Example: Expected 4.0s, accepts 2.8-5.2s (±30%)

---

### Review Queue Thresholds

```yaml
thresholds:
  review_queue:
    # Priority calculation weights
    recency_weight: 0.4
    simplicity_weight: 0.3
    impact_weight: 0.3

    # Priority thresholds
    high_priority_threshold: 0.7
    low_priority_threshold: 0.4

    # Auto-escalation
    max_time_in_queue: 7200  # seconds (2 hours)
    escalation_action: "llm_review"  # or "human_review"
```

**See:** L4_ReviewQueue.md for priority calculation details

---

## Alert Configuration

### Alert Channels

```yaml
alerts:
  channels:
    - type: "log"
      enabled: true
      level: "warning"  # info, warning, critical

    - type: "email"
      enabled: true
      level: "critical"
      to: !secret admin_email
      smtp:
        host: !secret smtp_host
        port: 587
        username: !secret smtp_username
        password: !secret smtp_password

    - type: "webhook"
      enabled: false
      url: !secret webhook_url
      level: "warning"
```

### Alert Thresholds

```yaml
alerts:
  thresholds:
    # Processing lag
    processing_lag_warning: 15  # seconds
    processing_lag_critical: 30

    # GPU utilization
    gpu_saturation_threshold: 95  # percent
    gpu_underutilized_threshold: 50

    # Queue depths
    merge_queue_warning: 50  # tracks
    merge_queue_critical: 100
    review_queue_warning: 20
    review_queue_critical: 50

    # Auto-merge rate
    auto_merge_rate_warning: 0.60  # 60%
    auto_merge_rate_target: 0.80  # 80%

    # Database performance
    db_query_warning: 100  # milliseconds
    db_query_critical: 500
```

**See:** L3_SystemHealth.md for alert logic

---

## Operational Modes

### Online Learning Mode

**Purpose:** Conservative operation during bootstrap phase

```yaml
mode: "online_learning"

thresholds:
  identity_resolution:
    face_match_threshold: 0.85  # Higher (more conservative)

  review_queue:
    auto_merge_enabled: false  # All uncertain → queue
    llm_review_enabled: true
    human_validation_required: true
```

**Characteristics:**
- High face match threshold (0.85)
- All uncertain tracks queued for review
- Human validation required for training data
- No auto-merge without strong evidence

**Duration:** First 2-4 weeks (bootstrap phase)

---

### Autonomous Mode

**Purpose:** High-autonomy operation after system trained

```yaml
mode: "autonomous"

thresholds:
  identity_resolution:
    face_match_threshold: 0.70  # Lower (more autonomous)

  review_queue:
    auto_merge_enabled: true  # High-confidence auto-merge
    llm_review_enabled: true
    human_validation_required: false
```

**Characteristics:**
- Lower face match threshold (0.70)
- Auto-merge enabled for high-confidence decisions
- LLM reviews most of queue
- Human validation optional (only for complex cases)

**Transition criteria:**
- Auto-merge rate > 80%
- Grid learning > 80% complete
- Face library > 100 validated samples per agent

---

## External AI Configuration

### CodeProject.AI (Face Recognition)

```yaml
external_ai:
  face_recognition:
    enabled: true
    api_url: !secret codeproject_url
    api_key: !secret codeproject_key

    # Face library management
    library_path: "/data/face_library"
    auto_register_threshold: 0.95  # Auto-add face if ≥ 0.95
    training_mode: "incremental"  # or "batch"

    # Request settings
    timeout: 5  # seconds
    retry_attempts: 3
    batch_size: 10  # faces per batch
```

**See:** L13_FaceRecognition.md for integration details

---

### LLM (Review Adjudication)

```yaml
external_ai:
  llm:
    enabled: true
    provider: "anthropic"  # or "openai"
    model: "claude-sonnet-4.5"
    api_key: !secret llm_api_key

    # Review settings
    max_tokens: 1000
    temperature: 0.0  # Deterministic
    vision_enabled: true

    # Batch processing
    batch_size: 10  # tracks per batch
    batch_interval: 300  # seconds (5 minutes)
```

---

## Storage Configuration

### File Storage

```yaml
storage:
  # Video frames
  frames:
    base_path: "/data/frames"
    format: "jpeg"
    quality: 85
    retention_days: 30  # Auto-delete old frames

  # Face crops
  faces:
    base_path: "/data/faces"
    format: "jpeg"
    quality: 95
    retention_days: 365  # Keep longer for training

  # Logs
  logs:
    base_path: "/var/log/marengocam"
    retention_days: 90
    rotation: "daily"  # or "size"
```

### Database Configuration

```yaml
database:
  type: "postgresql"
  host: "localhost"
  port: 5432
  database: "marengocam"
  username: !secret db_username
  password: !secret db_password

  # Connection pool
  pool_size: 10
  max_overflow: 20

  # Performance
  query_timeout: 30  # seconds
  index_optimization: true
```

---

## Hot-Reloading

### Configuration Watcher

**Process:**
1. Background thread monitors `marengo.yaml` (check every 5s)
2. On file modification, load and validate new config
3. Compute diff between new and current config
4. Apply changes gracefully to components
5. If errors spike (> 20% increase), rollback to last good config

**Graceful application:**
- **Thresholds:** Applied immediately (next decision cycle)
- **Camera settings:** Applied on next chunk (no interruption)
- **External AI:** Drain requests, switch to new endpoints
- **Alerts:** Update channels, re-subscribe

**Safety:**
- Validation before apply (schema check)
- Automatic rollback on error spike
- Critical alert if rollback triggered

---

## Secrets Management

### Secrets File

**File:** `secrets.yaml` (encrypted on disk)

**Example:**
```yaml
# Encrypted secrets file
driveway_main_url: "rtsp://192.168.1.10:554/stream1"
driveway_sub_url: "rtsp://192.168.1.10:554/stream2"
admin_email: "admin@example.com"
smtp_host: "smtp.gmail.com"
smtp_username: "marengo@example.com"
smtp_password: "app-specific-password"
codeproject_url: "http://localhost:32168"
codeproject_key: "abc123def456"
llm_api_key: "sk-ant-..."
db_username: "marengocam"
db_password: "secure_password"
```

**Encryption:**
- Windows: DPAPI (Data Protection API)
- Cross-platform: AES-256-GCM

**Excluded from version control:** `.gitignore` must include `secrets.yaml`

---

### Placeholder Syntax

**In `marengo.yaml`:**
```yaml
cameras:
  - id: "Driveway"
    streams:
      main:
        url: !secret driveway_main_url  # Placeholder
```

**Runtime resolution:**
- Config loader reads `secrets.yaml`
- Decrypts file
- Injects plaintext secrets into config (in-memory only)
- Plaintext never written to disk

---

### CLI Management

**Commands:**
```bash
# Set a secret
marengocam secrets set driveway_main_url "rtsp://..."

# Get a secret (masked)
marengocam secrets get driveway_main_url
# Output: "rtsp://192.168.1.10:554/st****"

# List all secrets (keys only)
marengocam secrets list

# Delete a secret
marengocam secrets delete driveway_main_url
```

**See:** L14_ConfigCLI.py for implementation

---

## Configuration Validation

### Schema Validation

**On load, validate:**
- Required fields present (camera IDs, stream URLs)
- Data types correct (numbers, strings, booleans)
- Enum values valid (camera states, alert levels)
- Ranges valid (thresholds 0.0-1.0, FPS > 0)

**Validation library:** `pydantic` or `jsonschema`

**On validation failure:**
- Reject config
- Log error with details
- Keep current config active
- Send critical alert

---

### Cross-Field Validation

**Check logical consistency:**
- Portal `camera_id` references existing camera
- Grid cells within bounds (0-5 for 6×6 grid)
- Face threshold < auto-register threshold
- Alert thresholds: warning < critical

**Example inconsistency:**
```yaml
# INVALID: warning > critical
alerts:
  thresholds:
    processing_lag_warning: 30
    processing_lag_critical: 15  # ERROR: should be > 30
```

---

## Configuration UI

**See:** L4_ConfigurationUI.md for UI specification

**Key features:**
- Add/edit/delete cameras
- Configure portals (drag-and-drop on grid)
- Adjust thresholds (sliders with live preview)
- Manage alert channels
- View/edit secrets (masked)

**Access:** Web UI (admins only) + config files (advanced users)

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - Configuration component overview

### Other L3 Components
- **L3_ChunkProcessing.md** - Camera states, thresholds
- **L3_IdentityResolution.md** - Portal handling, merging thresholds
- **L3_SystemHealth.md** - Alert thresholds
- **L3_Gui.md** - Configuration tool overview

### L4 Concepts
- **L4_ConfigurationUI.md** (planned) - Configuration tool UI specification

### Implementation
- **L14_ConfigLoader.py** - YAML loading, secrets injection
- **L14_ConfigValidator.py** - Schema validation
- **L14_ConfigWatcher.py** - Hot-reloading, rollback
- **L14_ConfigCLI.py** - CLI for secrets management
