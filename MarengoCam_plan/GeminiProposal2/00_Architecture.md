# 00 - System Architecture & Resilience

## Purpose
Defines the physical and logical layout of MarengoCam, with a primary focus on **Resilience**, **Resource Management**, and **Service Boundaries**.

## 1. High-Level Topology

### 1.1. The "Hub and Spoke" Data Flow
- **Hub:** PostgreSQL (State) + Filesystem (Media). The database runs in its own container and is managed by the Docker Compose supervisor.
- **Spokes:** Independent Python processes (Services).
- **Bus:** Redis (Optional, for fast IPC) or Polling (Simpler).

**Principle:** Services **never** talk directly to each other. They talk to the Database.
- *Bad:* `ChunkProcessor` sends HTTP request to `IdentityResolver`.
- *Good:* `ChunkProcessor` writes `LocalTrack` to DB. `IdentityResolver` reads it.
- *Why:* If `IdentityResolver` crashes, `ChunkProcessor` keeps recording.

## 2. Service Boundaries (Docker Containers)

| Service | Responsibility | Criticality | VRAM Budget |
| :--- | :--- | :--- | :--- |
| **db** | PostgreSQL + pgvector | **Critical** | N/A (RAM) |
| **ingest** | `ChunkProcessor` (One per camera or pooled) | **Critical** | 2GB (YOLO) |
| **brain** | `IdentityResolver` | High | 0GB (CPU) |
| **face_api** | CodeProject.AI Server | Medium | 4GB (Shared) |
| **web** | Dashboard & API | Medium | N/A |
| **janitor** | `SystemHealth` | Low | N/A |

## 3. Resilience Patterns

### 3.1. The "Circuit Breaker" (AI Dependencies)
The system must function (degraded) even if AI is dead.
- **Scenario:** CodeProject.AI crashes or hangs.
- **Mechanism:** `ChunkProcessor` wraps AI calls in a 500ms timeout.
- **Fallback:** If timeout/error:
    - Log error to `system_health`.
    - Save `LocalTrack` *without* Face Embedding.
    - `IdentityResolver` relies purely on Spatial/Temporal logic.
- **Recovery:** Retry connection every 30s.

### 3.2. "Survival Mode" (Resource Exhaustion)
Defined in `04_SystemHealth`, but architecturally enforced here.
- **Trigger:** Disk > 90% OR Ingest Lag > 5s.
- **Action:**
    - **Load Shedding:** Drop B-frames. Skip Face Recognition.
    - **Cannibalization:** Aggressively delete oldest footage (ignoring retention policy).
- **Goal:** Never stop recording the *current* moment.

### 3.3. Process Isolation (The "Watchdog")
- **Supervisor:** Docker Compose `restart: always`.
- **Health Check:** Each service exposes `GET /health`.
- **Logic:** If `ingest` hangs (GPU lockup), Docker kills and restarts it.
- **State Safety:** Since state is in DB, restart is safe. It just resumes the next video chunk.

## 4. Resource Management (The "Metal")

### 4.1. GPU VRAM Budgeting (RTX 3060/4060 - 8GB Target)
- **Reserved (System):** 1GB
- **CodeProject.AI (Face/Plate):** 3GB (Pinned)
- **YOLO (Object Detection):** 3GB (Pinned via Torch backend)
- **Buffer:** 1GB
- **Enforcement:** `CUDA_VISIBLE_DEVICES` and PyTorch memory fraction limits in `docker-compose.yml`.

### 4.2. Concurrency Control
- **Database:** Connection pooling (pgbouncer) to prevent `ingest` threads from starving the `web` UI.
- **Disk I/O:** `ingest` writes are buffered. `janitor` deletes are throttled (nice/ionice) to avoid IO wait spikes.

## 5. Automation & Event Bus

### 5.1. The "Rule Engine"
- **Location:** Inside `IdentityResolver` (post-merge) or a dedicated `AutomationService`.
- **Trigger:** Database Change (CDC) or Polling.
- **Logic:**
    ```python
    if event.type == "person_confirmed" and event.zone == "backyard":
        mqtt.publish("home/security/backyard", "person")
    ```

### 5.2. External Notifications (MQTT)
- **Broker:** Mosquitto (Optional).
- **Payload:** JSON.
- **QoS:** Level 0 (Fire and forget). We don't block the pipeline for a notification.

## 6. Development & Debugging (The "Time Machine")

### 6.1. Replay Harness
- **Concept:** Ability to feed *past* video chunks into the *current* code.
- **Mechanism:** `ingest` service accepts `--source=file://...` instead of RTSP.
- **Use Case:** "Why did we miss the mailman?" -> Re-run that hour of video with `DEBUG` logging enabled.
