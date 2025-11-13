# Configuration System Design

This document defines the complete configuration architecture for MarengoCam, including operational stages, external AI service management, resource throttling, and hot-reload capabilities.

---

## Philosophy

**Multi-Stage Flexibility:**
- Development vs Production environments need different resource profiles
- Phases 0-2 can run on underpowered hardware with throttling
- Later phases (3-5) need full performance for real-time merging

**Service Discovery & Health:**
- Multiple CodeProject.AI servers can run simultaneously (different machines, GPU vs CPU)
- System tracks response times, queue depth, CPU state per service
- Automatic load balancing and failover based on health metrics

**Observable & Debuggable:**
- All thresholds tunable at runtime with hot-reload
- Config changes versioned and stamped on artifacts
- Rollback capability if config change degrades metrics

---

## Operational Stages

Stages control resource usage, inference rates, and pipeline complexity. Think of stages as "development gears" separate from architectural Phases.

### **Stage: DEV_SLOW** (Current need for next few days)
*For underpowered development machines; proves logic without taxing resources.*

```yaml
stage: DEV_SLOW

acquisition:
  frame_rate: 2.0                    # FPS per camera (vs 10 FPS production)
  cameras_concurrent: 5              # Process 5 at a time (vs all 20)
  jpeg_quality: 75                   # Lower quality = smaller bandwidth
  
detection:
  yolo_batch_size: 1                 # Process one frame at a time
  yolo_threads: 2                    # Limit CPU usage
  confidence_threshold: 0.6          # Higher = fewer detections to track
  
tracking:
  max_tracks_per_camera: 10          # Limit memory
  update_interval: 1.0               # Update tracks every 1s vs 0.1s
  
merging:
  enabled: false                     # Skip cross-camera merging entirely
  panel_generation: false            # Skip expensive panel assembly
  
external_ai:
  face_recognition:
    enabled: false                   # Defer face recognition
  lpr:
    enabled: false                   # Defer LPR
    
storage:
  frames_retention_hours: 24         # Keep only 1 day vs 9 days
  write_batch_interval: 30.0         # Batch writes every 30s vs 5s
```

**Capabilities:**
- ✅ Frame acquisition and JPEG storage
- ✅ Single-camera detection and tracking (32×32 grid)
- ✅ Height calibration per cell
- ✅ Basic camera state machine (Standby/Armed/Active)
- ❌ Cross-camera merging disabled
- ❌ Face recognition disabled
- ❌ LPR disabled

**Purpose:** Prove Phase 0-1 logic (acquisition, detection, intra-camera tracking) without melting laptop.

---

### **Stage: DEV_FULL**
*Full development system on capable hardware; all features enabled but with shorter retention.*

```yaml
stage: DEV_FULL

acquisition:
  frame_rate: 10.0                   # Full 10 FPS
  cameras_concurrent: 20             # All cameras
  jpeg_quality: 85
  
detection:
  yolo_batch_size: 4                 # Batch processing
  yolo_threads: 4
  confidence_threshold: 0.5
  
tracking:
  max_tracks_per_camera: 50
  update_interval: 0.1               # Fast updates
  
merging:
  enabled: true                      # Cross-camera merging active
  panel_generation: true
  llm_calls_enabled: true            # Gemini adjudication
  auto_merge_threshold: 0.80         # High confidence auto-merge
  
external_ai:
  face_recognition:
    enabled: true
    models: ["facerec-high"]         # Full accuracy model
  lpr:
    enabled: true
    cameras: ["FrontGate", "DrivewayEast"]  # Select cameras only
    
storage:
  frames_retention_hours: 72         # 3 days vs 9 days production
  write_batch_interval: 5.0
```

**Capabilities:**
- ✅ Full acquisition and storage
- ✅ Intra-camera and cross-camera tracking
- ✅ Face recognition (select cameras)
- ✅ LPR (select cameras)
- ✅ Panel generation and LLM adjudication
- ⚠️ Shorter retention (3 days vs 9 days)

**Purpose:** Full Phase 0-3 testing on development machine before production rollout.

---

### **Stage: PROD_LEARNING**
*Production system in learning mode; high retention, conservative auto-merge, human review queue active.*

```yaml
stage: PROD_LEARNING

acquisition:
  frame_rate: 10.0
  cameras_concurrent: 20
  jpeg_quality: 90                   # Higher quality for face/plate
  
detection:
  yolo_batch_size: 8                 # Maximize GPU utilization
  yolo_threads: 8
  confidence_threshold: 0.45         # Catch more marginal detections
  
tracking:
  max_tracks_per_camera: 100
  update_interval: 0.1
  
merging:
  enabled: true
  panel_generation: true
  llm_calls_enabled: true
  auto_merge_threshold: 0.85         # Conservative; prefer human review
  human_review_queue: true           # Queue medium-confidence merges
  
external_ai:
  face_recognition:
    enabled: true
    models: ["facerec-high", "facerec-fast"]  # Multiple models for comparison
  lpr:
    enabled: true
    cameras: ["FrontGate", "DrivewayEast", "GarageEntry"]
    
storage:
  frames_retention_hours: 216        # Full 9 days
  metadata_retention: indefinite
  write_batch_interval: 5.0
  grid_backup_interval_hours: 168    # Weekly grid backups
```

**Capabilities:**
- ✅ Full production feature set
- ✅ Human review queue for learning
- ✅ Weekly grid backups
- ✅ Full 9-day retention
- ⚠️ Conservative auto-merge (needs human validation)

**Purpose:** Phase 3-4 with human-in-loop learning; build face library and edge confidence.

---

### **Stage: PROD_AUTO**
*Mature production; high auto-merge rate, minimal human review.*

```yaml
stage: PROD_AUTO

# Same as PROD_LEARNING except:

merging:
  auto_merge_threshold: 0.70         # Aggressive auto-merge
  human_review_queue: false          # Only queue on request/error
  llm_calls_enabled: true            # Still use LLM for medium confidence
  
external_ai:
  face_recognition:
    auto_register_threshold: 0.75    # Auto-add new faces to library
```

**Capabilities:**
- ✅ High auto-merge rate (70%+ automatic)
- ✅ Minimal human review (only failures/edge cases)
- ✅ Automatic face library expansion

**Purpose:** Phase 5 autonomous operation after learning stabilizes.

---

## External AI Service Configuration

### Service Registry (Multi-Instance Support)

Each CodeProject.AI instance registered with URL, priority, enabled models, and health tracking.

```yaml
external_ai:
  services:
    - name: "cpai-gpu-desktop"
      url: "http://192.168.1.100:32168"
      priority: 1                      # Lower = higher priority
      enabled: true
      health_check_interval_s: 30
      
      models:
        face_recognition:
          enabled: true
          endpoint: "/v1/vision/face/recognize"
          model_name: "facerec-high"   # High accuracy, slower
          max_qps: 10                  # Queries per second limit
          timeout_ms: 300
          
        lpr:
          enabled: true
          endpoint: "/v1/vision/lpr"
          model_name: "alpr"
          max_qps: 5
          timeout_ms: 400
          
    - name: "cpai-cpu-laptop"
      url: "http://localhost:32168"
      priority: 2                      # Fallback service
      enabled: true
      health_check_interval_s: 30
      
      models:
        face_recognition:
          enabled: true
          endpoint: "/v1/vision/face/recognize"
          model_name: "facerec-fast"   # Fast CPU model
          max_qps: 3
          timeout_ms: 500
          
        lpr:
          enabled: false               # No LPR on CPU (too slow)
```

**Multi-Server Strategy:**
- **Priority routing:** Route requests to lowest-priority healthy service first
- **Automatic failover:** If service times out or errors, try next priority
- **Load balancing:** If multiple services same priority, round-robin based on queue depth
- **Health tracking:** Disable service after N consecutive failures; re-enable after cooldown

---

### Service Health Metrics

Scheduler tracks these metrics per service (rolling 5-minute windows):

```python
@dataclass
class ServiceHealth:
    service_name: str
    url: str
    enabled: bool
    
    # Response time metrics (ms)
    response_time_p50: float
    response_time_p95: float
    response_time_p99: float
    
    # Throughput
    requests_per_second: float
    queue_depth: int                 # Pending requests
    
    # Reliability
    success_rate: float              # 0.0 to 1.0
    consecutive_errors: int
    last_error_ts: Optional[datetime]
    
    # Resource usage (if available via /status endpoint)
    cpu_percent: Optional[float]
    gpu_percent: Optional[float]
    memory_mb: Optional[int]
    
    # Circuit breaker state
    circuit_state: str               # CLOSED, OPEN, HALF_OPEN
    circuit_open_until: Optional[datetime]
```

**Scheduler Decision Logic:**

1. **Filter:** Exclude services where `circuit_state == OPEN` or `enabled == false`
2. **Sort:** By priority (ascending), then by `queue_depth` (ascending)
3. **Check capacity:** If `requests_per_second >= max_qps`, skip to next service
4. **Route:** Send request to first available service
5. **Update metrics:** Record latency, success/failure
6. **Circuit breaker:** After 5 consecutive errors, open circuit for 60s cooldown

---

### Model Selection (Per-Camera Overrides)

Some cameras may benefit from different models (e.g., fast model on low-priority cameras, high-accuracy on critical zones).

```yaml
cameras:
  FrontGate:
    face_recognition:
      model_preference: "facerec-high"  # Critical entry point
      min_confidence: 0.75
    lpr:
      enabled: true
      model_preference: "alpr"
      min_confidence: 0.70
      
  BackYard:
    face_recognition:
      model_preference: "facerec-fast"  # Low priority zone
      min_confidence: 0.65
    lpr:
      enabled: false                    # No vehicles expected
      
  DrivewayEast:
    face_recognition:
      enabled: false                    # Vehicle-only camera
    lpr:
      enabled: true
      model_preference: "alpr"
```

**Fallback Logic:**
- If preferred model unavailable, use any available model for that task
- If no services available, skip external AI for that request (system still functional)

---

## Resource Throttling (Stage-Aware)

### Frame Acquisition Throttle

```yaml
acquisition:
  frame_rate: 2.0                    # Target FPS per camera
  cameras_concurrent: 5              # Max cameras processed simultaneously
  
  # Adaptive throttling based on CPU
  cpu_throttle:
    enabled: true
    cpu_threshold_percent: 80        # If CPU > 80%, reduce frame rate
    throttle_step: 0.5               # Reduce by 0.5 FPS
    min_frame_rate: 0.5              # Never go below 0.5 FPS
```

**Behavior:**
- Monitor system CPU every 10s
- If CPU > threshold for 30s, reduce `frame_rate` by `throttle_step`
- If CPU < threshold - 10% for 60s, increase `frame_rate` by `throttle_step/2`
- Respect `min_frame_rate` and stage-defined max

---

### Detection Batch Throttle

```yaml
detection:
  yolo_batch_size: 1                 # Process N frames at once
  yolo_threads: 2                    # OpenCV thread count
  
  # Queue depth limits
  max_pending_frames: 50             # Drop frames if queue exceeds
  drop_strategy: "oldest"            # Drop oldest frames first
```

**Behavior:**
- If detection queue depth > `max_pending_frames`, drop frames per strategy
- Log dropped frame count to health metrics
- Alert if drop rate > 10% sustained

---

### External AI Throttle

```yaml
external_ai:
  global_qps_limit: 20               # Total queries/sec across all services
  per_service_qps_limit: 10          # Per service limit
  
  # Request prioritization
  priority_queue:
    enabled: true
    face_priority: 2                 # Higher = higher priority
    lpr_priority: 1
```

**Behavior:**
- Token bucket per service (refills at `per_service_qps_limit` rate)
- If token unavailable, queue request (up to `max_queue_depth`)
- If queue full, drop lowest-priority requests

---

## Configuration Hot-Reload

All configuration changes applied without restart:

```yaml
hot_reload:
  enabled: true
  config_file: "config/marengo.yaml"
  watch_interval_s: 5                # Check for changes every 5s
  
  # Version tracking
  version_stamping:
    enabled: true
    stamp_artifacts: true            # Add config_version to all artifacts
    
  # Rollback safety
  validation:
    enabled: true                    # Validate before applying
    dry_run_duration_s: 60           # Test for 60s before committing
    auto_rollback_on_errors: true    # Rollback if error rate spikes
```

**Hot-Reload Flow:**
1. Detect `config/marengo.yaml` change (file modification time)
2. Parse and validate new config (schema check, range validation)
3. Compute diff: what changed?
4. Apply changes to running pipeline:
   - Thresholds: immediate effect
   - External AI services: graceful drain of old service, route to new
   - Frame rate: adjust acquisition loop target
5. Stamp all new artifacts with `config_version` (SHA256 of config file)
6. Monitor metrics for 60s; if error rate increases >20%, auto-rollback

**Rollback Command:**
```bash
marengo config rollback --version <config_sha256>
```

---

## Configuration File Structure

```yaml
# config/marengo.yaml

version: "1.0.0"
stage: "DEV_SLOW"  # DEV_SLOW | DEV_FULL | PROD_LEARNING | PROD_AUTO

# Blue Iris connection
blue_iris:
  host: "http://192.168.1.50"
  port: 81
  username: !secret bi_username
  password: !secret bi_password
  timeout_ms: 1000

# Frame acquisition
acquisition:
  frame_rate: 2.0
  cameras_concurrent: 5
  jpeg_quality: 75
  cpu_throttle:
    enabled: true
    cpu_threshold_percent: 80
    throttle_step: 0.5
    min_frame_rate: 0.5

# YOLO detection
detection:
  model_path: "models/yolov8s.pt"
  yolo_batch_size: 1
  yolo_threads: 2
  confidence_threshold: 0.6
  max_pending_frames: 50
  drop_strategy: "oldest"
  
  classes:
    person: {enabled: true, confidence: 0.6}
    vehicle: {enabled: true, confidence: 0.5}  # Cars easier to detect
    animal: {enabled: true, confidence: 0.65}

# Intra-camera tracking
tracking:
  max_tracks_per_camera: 10
  update_interval: 1.0
  iou_threshold: 0.3
  max_age_seconds: 3.0
  
  # 32x32 grid
  intra_grid:
    rows: 32
    cols: 32
    height_calibration:
      min_samples: 20
      update_alpha: 0.1

# Cross-camera merging
merging:
  enabled: false  # Disabled in DEV_SLOW
  panel_generation: false
  llm_calls_enabled: false
  auto_merge_threshold: 0.80
  human_review_queue: false
  
  # 8x8 inter-camera grid
  inter_grid:
    rows: 8
    cols: 8
    downsampling_ratio: 4
    edge_learning:
      alpha: 0.3
      min_support: 3
      class_agnostic: true

# External AI services
external_ai:
  global_qps_limit: 20
  
  services:
    - name: "cpai-gpu-desktop"
      url: "http://192.168.1.100:32168"
      priority: 1
      enabled: true
      health_check_interval_s: 30
      
      models:
        face_recognition:
          enabled: true
          endpoint: "/v1/vision/face/recognize"
          model_name: "facerec-high"
          max_qps: 10
          timeout_ms: 300
          
        lpr:
          enabled: true
          endpoint: "/v1/vision/lpr"
          model_name: "alpr"
          max_qps: 5
          timeout_ms: 400
          
    - name: "cpai-cpu-laptop"
      url: "http://localhost:32168"
      priority: 2
      enabled: true
      health_check_interval_s: 30
      
      models:
        face_recognition:
          enabled: true
          endpoint: "/v1/vision/face/recognize"
          model_name: "facerec-fast"
          max_qps: 3
          timeout_ms: 500
          
        lpr:
          enabled: false

# LLM integration (Gemini Flash 2.0)
llm:
  enabled: false  # Disabled in DEV_SLOW
  provider: "gemini"
  model: "gemini-2.0-flash-exp"
  api_key: !secret gemini_api_key
  timeout_ms: 5000
  max_retries: 2
  panel_size: 768

# Storage
storage:
  base_path: "C:/MarengoCam/storage"
  frames_retention_hours: 24  # 1 day in DEV_SLOW
  metadata_retention: "indefinite"
  write_batch_interval: 30.0
  grid_backup_interval_hours: 168  # Weekly

# Per-camera overrides
cameras:
  FrontGate:
    enabled: true
    vehicle_eligible: true
    face_recognition:
      enabled: true
      model_preference: "facerec-high"
      min_confidence: 0.75
    lpr:
      enabled: true
      model_preference: "alpr"
      min_confidence: 0.70
      
  BackYard:
    enabled: true
    vehicle_eligible: false
    face_recognition:
      enabled: true
      model_preference: "facerec-fast"
      min_confidence: 0.65
    lpr:
      enabled: false

# Hot-reload
hot_reload:
  enabled: true
  config_file: "config/marengo.yaml"
  watch_interval_s: 5
  version_stamping:
    enabled: true
    stamp_artifacts: true
  validation:
    enabled: true
    dry_run_duration_s: 60
    auto_rollback_on_errors: true

# Secrets (external file)
secrets_file: "config/secrets.yaml"  # Encrypted with DPAPI or AES
```

---

## Secrets Management

Secrets stored separately and encrypted:

```yaml
# config/secrets.yaml (encrypted on disk)

bi_username: "admin"
bi_password: "BlueIrisSecurePass123"
gemini_api_key: "AIzaSy..."
```

**Encryption:**
- Windows: DPAPI (Data Protection API) - user-scoped encryption
- Cross-platform: AES-256-GCM with key derived from machine ID + user passphrase

**Usage in config:**
```yaml
blue_iris:
  username: !secret bi_username  # Placeholder; resolved at runtime
  password: !secret bi_password
```

**CLI:**
```bash
# Set secret (prompts for value)
marengo secrets set bi_password

# Reveal secret (with timer)
marengo secrets reveal bi_password --timeout 30s

# Test connection (validates secrets work)
marengo secrets test blue_iris
```

---

## Configuration Validation

Schema enforced at load time:

```python
from pydantic import BaseModel, Field, validator

class AcquisitionConfig(BaseModel):
    frame_rate: float = Field(gt=0.0, le=10.0)
    cameras_concurrent: int = Field(ge=1, le=50)
    jpeg_quality: int = Field(ge=50, le=100)
    
    @validator('frame_rate')
    def validate_frame_rate(cls, v, values):
        if 'stage' in values and values['stage'] == 'DEV_SLOW':
            assert v <= 5.0, "DEV_SLOW stage limited to 5 FPS"
        return v

class ServiceConfig(BaseModel):
    name: str
    url: str = Field(regex=r'^https?://')
    priority: int = Field(ge=1, le=10)
    enabled: bool = True
    
    @validator('url')
    def validate_url_reachable(cls, v):
        # Optional: ping URL during validation
        return v
```

**Validation Errors:**
- Log detailed error with line number and suggestion
- Refuse to start if config invalid
- During hot-reload, reject invalid config and keep old config active

---

## Health Dashboard

Real-time metrics viewable via web UI or CLI:

```bash
marengo health status

# Output:
Stage: DEV_SLOW
Uptime: 2h 34m

Acquisition:
  Frame Rate: 2.0 FPS (target: 2.0)
  Cameras Active: 5/5
  CPU Usage: 42% (throttle threshold: 80%)
  Dropped Frames: 0

Detection:
  Queue Depth: 3 frames
  Inference Time: 45ms (p50), 78ms (p95)
  Detections/sec: 12

External AI Services:
  cpai-gpu-desktop [HEALTHY]
    Priority: 1
    Response Time: 124ms (p50), 234ms (p95)
    Requests/sec: 4.2
    Queue Depth: 0
    Success Rate: 99.8%
    Circuit: CLOSED
    
  cpai-cpu-laptop [HEALTHY]
    Priority: 2
    Response Time: 312ms (p50), 487ms (p95)
    Requests/sec: 1.1
    Queue Depth: 0
    Success Rate: 100%
    Circuit: CLOSED

Storage:
  Frames Stored: 8,432 (1.2 GB)
  Retention: 24 hours
  Write Latency: 12ms (p50)
```

---

## Migration Path (Your Current Situation)

**Week 1-2: DEV_SLOW on Laptop**
```yaml
stage: DEV_SLOW
frame_rate: 2.0
cameras_concurrent: 5
merging.enabled: false
external_ai.services: []  # No CodeProject.AI yet
```

**Goals:**
- Prove acquisition + detection + single-camera tracking
- Build height calibration per camera
- Test configuration hot-reload

---

**Week 3: Add CodeProject.AI (Low Priority)**
```yaml
stage: DEV_SLOW
external_ai:
  services:
    - name: "cpai-cpu-laptop"
      url: "http://localhost:32168"
      priority: 1
      enabled: true
      models:
        face_recognition:
          enabled: true
          model_name: "facerec-fast"
          max_qps: 2  # Very limited
```

**Goals:**
- Test face recognition integration
- Measure latency on underpowered hardware
- Build failure handling (circuit breaker, timeouts)

---

**Week 4+: Upgrade to DEV_FULL (Better Hardware)**
```yaml
stage: DEV_FULL
frame_rate: 10.0
cameras_concurrent: 20
merging.enabled: true
external_ai:
  services:
    - name: "cpai-gpu-desktop"
      url: "http://192.168.1.100:32168"
      priority: 1
      models:
        face_recognition:
          model_name: "facerec-high"
          max_qps: 10
```

**Goals:**
- Full cross-camera merging (Phase 2)
- Panel generation and LLM adjudication
- Edge learning and grid backups

---

## Open Items

1. **CodeProject.AI Discovery:**
   - Auto-discover services on local network via mDNS/Bonjour?
   - Or require manual URL registration?

2. **GPU Affinity:**
   - If multiple GPUs available, pin YOLO to GPU 0, CodeProject.AI to GPU 1?
   - Or rely on CodeProject.AI's internal scheduling?

3. **Stage Transitions:**
   - Automatic stage promotion based on metrics (e.g., DEV_FULL → PROD_LEARNING when auto-merge rate >50%)?
   - Or manual-only transitions?

4. **Per-Camera Resource Budgets:**
   - High-priority cameras get more frame rate budget than low-priority?
   - Dynamic reallocation based on activity?

5. **Configuration UI:**
   - Web-based config editor (like Grafana provisioning)?
   - Or CLI + YAML editing only?

---

## Summary

**Stages** control resource usage and feature complexity (DEV_SLOW → DEV_FULL → PROD_LEARNING → PROD_AUTO).

**External AI Services** registered with URL, priority, model checkboxes; scheduler tracks response times, queue depth, CPU state for intelligent routing.

**Resource Throttling** adapts frame rate and batch size based on CPU usage; protects underpowered hardware.

**Hot-Reload** enables runtime config changes without restart; versioned and rollback-safe.

**Your Current Path:** Start with DEV_SLOW (2 FPS, 5 cameras, no merging) to develop Phases 0-1 on underpowered laptop, then scale up as hardware improves.
