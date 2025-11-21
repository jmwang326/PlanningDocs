# 04 - Component: System Health (The Immune System)

## Purpose
Monitors the physical and logical health of the system. It is responsible for **Observability** (Metrics) and **Survival** (Retention & Load Shedding).

**Philosophy:**
- **Self-Healing:** Attempt to fix problems (restart services, delete old files) before alerting the human.
- **Prioritized Survival:** Recent events > Old events. Metadata > Video.

---

## 1. Interface Contract

### 1.1. Primary Functions
```python
def check_health() -> HealthStatus:
    """
    Runs every 60s.
    Checks: Disk Space, GPU Temp, Queue Depth, Service Liveness.
    Output: Writes to `system_health` table.
    """

def enforce_retention(policy: RetentionPolicy):
    """
    Runs every 1 hour (or on 'Low Disk' trigger).
    Action: Deletes files to satisfy policy.
    """
```

### 1.2. Evolutionary Stages

**Stage 1: The Observer (Passive)**
- **Logic:** Collects metrics and logs them.
- **Goal:** Establish baseline performance (FPS, VRAM usage).

**Stage 2: The Janitor (Active)**
- **Logic:** Deletes files older than X days.
- **Goal:** Prevent disk fill-up under normal operation.

**Stage 3: The Survivalist (Reactive)**
- **Logic:** Implements "Load Shedding" and "Emergency Deletion" when thresholds are breached.

---

## 2. Monitoring Metrics

| Metric | Source | Threshold (Warning) | Threshold (Critical) |
| :--- | :--- | :--- | :--- |
| **Disk Usage** | `psutil` | 80% | 90% |
| **Ingest Lag** | DB (Created vs Processed) | 10s | 60s |
| **GPU Mem** | `nvidia-smi` | 7GB | 7.8GB |
| **Queue Depth** | DB (`local_tracks` unmerged) | 50 | 200 |

---

## 3. Survival Mode Logic

### 3.1. Trigger: Disk Full (>90%)
**Action Plan (In Order):**
1.  **Delete "Trash" Video:** Raw chunks > 24h old that have NO associated `LocalTracks`.
2.  **Delete Old Video:** Raw chunks > retention_period (start deleting from oldest).
3.  **Delete Thumbnails:** Crops > 30 days.
4.  **Emergency:** Delete ALL raw video > 1h old. Keep only Metadata/Thumbnails.

### 3.2. Trigger: High Lag (>60s)
**Action Plan:**
1.  **Throttle Ingest:** Tell `ChunkProcessor` to skip Face Recognition.
2.  **Drop Frames:** Tell `ChunkProcessor` to process only 5 FPS.
3.  **Drop Chunks:** Discard every other chunk until caught up.

---

## 4. Maintenance Tasks (The Janitor)

- **Daily:** `VACUUM ANALYZE` PostgreSQL tables.
- **Daily:** Prune `audit_logs` > 1 year.
- **Weekly:** Re-index `grid_links` and `local_tracks`.
