# Level 3 - Configuration

## Purpose
How cameras, portals, thresholds, and system parameters are managed, validated, and hot-reloaded.

---

## Configuration Structure

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

```yaml
cameras:
  - id: "Driveway"
    enabled: true
    streams:
      main:
        url: !secret driveway_main_url
        resolution: "1920x1080"
        fps: 25
      sub:
        url: !secret driveway_sub_url
        resolution: "640x480"
        fps: 10
    grid:
      width: 6
      height: 6
    thresholds:
      detection_confidence: 0.5
      motion_threshold: 0.02
      min_track_duration: 3.0  # seconds
    initial_state: "Standby"
    armed_timeout: 30  # seconds
    post_roll: 15  # seconds
```

**Required:** `id`, `streams.sub.url`

**Camera states:**
- **Standby:** Motion gate, sparse YOLO, no frame persistence
- **Armed:** Moderate YOLO, accumulate candidates, save sub frames
- **Active:** Full YOLO, save sub + main frames
- **Post:** Moderate YOLO, save sub frames

---

## Portal Configuration

```yaml
portals:
  - id: "garage_door"
    camera_id: "Driveway"
    cells: [[3, 2], [3, 3], [4, 2], [4, 3]]
    type: "structure_entry"  # or "vehicle_entry"
    bidirectional: true
```

**Types:**
- **structure_entry:** Building entrance (time gap irrelevant)
- **vehicle_entry:** Garage/vehicle entry (person enters vehicle)

---

## Threshold Configuration

```yaml
thresholds:
  detection:
    person: 0.5
    vehicle: 0.6
    animal: 0.4
    motion_threshold: 0.02
    min_track_duration: 3.0

  identity_resolution:
    face_match_threshold: 0.75
    reid_similarity_threshold: 0.70
    overlap_zone_threshold: 0.4  # seconds
    grid_timing_variance: 0.3
    alibi_check_enabled: true
    alibi_threshold: 2  # Max alternative sources

  review_queue:
    recency_weight: 0.4
    simplicity_weight: 0.3
    impact_weight: 0.3
    high_priority_threshold: 0.7
    max_time_in_queue: 7200  # 2 hours
    escalation_action: "llm_review"
```

---

## Alert Configuration

```yaml
alerts:
  channels:
    - type: "log"
      enabled: true
      level: "warning"
    - type: "email"
      enabled: true
      level: "critical"
      to: !secret admin_email
      smtp:
        host: !secret smtp_host
        port: 587
        username: !secret smtp_username
        password: !secret smtp_password

  thresholds:
    processing_lag_warning: 15  # seconds
    processing_lag_critical: 30
    gpu_saturation_threshold: 95  # percent
    merge_queue_warning: 50
    merge_queue_critical: 100
    review_queue_warning: 20
    review_queue_critical: 50
    auto_merge_rate_warning: 0.60
    auto_merge_rate_target: 0.80
```

---

## Operational Modes

### Online Learning Mode

```yaml
mode: "online_learning"

thresholds:
  identity_resolution:
    face_match_threshold: 0.85  # Higher (conservative)
  review_queue:
    auto_merge_enabled: false
    llm_review_enabled: true
    human_validation_required: true
```

**Characteristics:** High face threshold, all uncertain → queue, human validation required.

**Duration:** First 2-4 weeks (bootstrap).

---

### Autonomous Mode

```yaml
mode: "autonomous"

thresholds:
  identity_resolution:
    face_match_threshold: 0.70  # Lower (autonomous)
  review_queue:
    auto_merge_enabled: true
    llm_review_enabled: true
    human_validation_required: false
```

**Characteristics:** Lower face threshold, auto-merge enabled, LLM reviews most queue.

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
    library_path: "/data/face_library"
    auto_register_threshold: 0.95
    training_mode: "incremental"
    timeout: 5  # seconds
    retry_attempts: 3
    batch_size: 10
```

---

### LLM (Review Adjudication)

```yaml
external_ai:
  llm:
    enabled: true
    provider: "anthropic"
    model: "claude-sonnet-4.5"
    api_key: !secret llm_api_key
    max_tokens: 1000
    temperature: 0.0
    vision_enabled: true
    batch_size: 10
    batch_interval: 300  # seconds
```

---

## Storage Configuration

```yaml
storage:
  frames:
    base_path: "/data/frames"
    format: "jpeg"
    quality: 85
    retention_days: 30
  faces:
    base_path: "/data/faces"
    quality: 95
    retention_days: 365
  logs:
    base_path: "/var/log/marengocam"
    retention_days: 90
```

---

## Database Configuration

```yaml
database:
  type: "postgresql"
  host: "localhost"
  port: 5432
  database: "marengocam"
  username: !secret db_username
  password: !secret db_password
  pool_size: 10
  max_overflow: 20
  query_timeout: 30
```

---

## Hot-Reloading

**Process:**
1. Monitor `marengo.yaml` (check every 5s)
2. On modification, load and validate new config
3. Compute diff between new and current
4. Apply changes gracefully
5. If errors spike (> 20% increase), rollback

**Graceful application:**
- **Thresholds:** Applied immediately (next decision cycle)
- **Camera settings:** Applied on next chunk
- **External AI:** Drain requests, switch endpoints
- **Alerts:** Update channels

**Safety:** Validation before apply, automatic rollback on error spike.

---

## Secrets Management

**File:** `secrets.yaml` (encrypted)

```yaml
driveway_main_url: "rtsp://192.168.1.10:554/stream1"
admin_email: "admin@example.com"
smtp_host: "smtp.gmail.com"
codeproject_url: "http://localhost:32168"
llm_api_key: "sk-ant-..."
db_username: "marengocam"
db_password: "secure_password"
```

**Encryption:** Windows DPAPI or AES-256-GCM

**Placeholder syntax:**
```yaml
streams:
  main:
    url: !secret driveway_main_url
```

**CLI commands:**
```bash
marengocam secrets set driveway_main_url "rtsp://..."
marengocam secrets get driveway_main_url  # Output: "rtsp://...st****"
marengocam secrets list
marengocam secrets delete driveway_main_url
```

---

## Configuration Validation

**On load, validate:**
- Required fields present
- Data types correct
- Enum values valid
- Ranges valid (thresholds 0.0-1.0, FPS > 0)

**Cross-field validation:**
- Portal `camera_id` references existing camera
- Grid cells within bounds (0-5 for 6×6)
- Face threshold < auto-register threshold
- Alert thresholds: warning < critical

**On failure:** Reject config, log error, keep current active, send critical alert.
