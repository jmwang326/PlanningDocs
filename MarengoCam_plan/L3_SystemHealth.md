# Level 3 - System Health (System Health Monitor)

## Purpose
Monitor performance, detect issues, alert on problems.

---

## Metrics to Monitor

### 1. Processing Lag

**What:** Time delay between video acquisition and LocalTrack creation

**Target:** < 15s

**Measurement:** `lag = now - (chunk_end_time + processing_time)`

**Alerts:**
- Warning: lag > 15s
- Critical: lag > 30s

**Causes:** GPU overloaded, too many active cameras, inference manager not prioritizing correctly.

---

### 2. GPU Utilization

**What:** Percentage of GPU capacity used for YOLO inference

**Target:** 70-90%

**Alerts:**
- Warning: > 95% (saturated, may drop frames)
- Warning: < 50% (underutilized)

**Causes:** Too many active cameras, inference rate miscalibrated, YOLO model too heavy.

---

### 3. Merge Queue Depth

**What:** Number of uncertain tracks waiting for identity resolution

**Target:** < 50 tracks

**Alerts:**
- Warning: > 50
- Critical: > 100

**Causes:** Face recognition service slow/down, too much concurrent activity, grid learning incomplete.

---

### 4. Review Queue Depth

**What:** Number of tracks queued for human/LLM review

**Target:** < 20 tracks

**Alerts:**
- Warning: > 20
- Critical: > 50

**Causes:** High ambiguity rate, LLM service slow/down, human not reviewing regularly.

---

### 5. Auto-Merge Rate

**What:** Percentage of tracks merged automatically (without human/LLM review)

**Target:** > 80% (after bootstrap)

**Alerts:**
- Warning: < 60%
- Info: > 90% (excellent!)

**Causes (low rate):** Face library incomplete, grid learning incomplete, high concurrent activity.

---

### 6. Frame Persistence Lag

**What:** Delay between frame acquisition and frame saved to disk

**Target:** < 5s

**Alerts:**
- Warning: > 5s
- Critical: > 10s

**Causes:** Disk I/O bottleneck, too many active cameras writing simultaneously, network latency.

---

### 7. Database Query Performance

**What:** Average time for common queries

**Target:** < 100ms

**Alerts:**
- Warning: > 100ms
- Critical: > 500ms

**Causes:** Missing indexes, too many LocalTracks (need archival), query inefficiency.

---

## Health Dashboard

**Sections:**

1. **Processing Status:** Current lag (per camera), GPU utilization, active cameras count

2. **Queue Status:** Merge queue depth, review queue depth, oldest unresolved track

3. **Performance Metrics:** Auto-merge rate (last hour), frame persistence lag, database query performance

4. **Alerts:** Warning-level issues, critical issues, recent alerts history

**Historical Trends:** Processing lag trends, GPU utilization patterns, auto-merge rate improvement, queue depth fluctuations.

**Visualization:** Time-series graphs (last 24 hours, last 7 days), heatmaps (by camera, by time of day).

---

## Alerting

**Levels:**
- **Info:** Positive indicators (auto-merge rate > 90%)
- **Warning:** Potential issues requiring attention (lag > 15s, queue depth > 50)
- **Critical:** Immediate action required (lag > 30s, GPU saturated, service down)

**Channels:** Dashboard (real-time), email (critical only), system log (all).

**Throttling:** Wait 5 minutes between same alert (don't spam).

---

## Automatic Remediation

**Self-healing actions:**

```python
if gpu_util > 95%:
    inference_manager.reduce_sampling_rate()  # Reduce Armed/Post sampling

if lag > 30s:
    inference_manager.prioritize_active_cameras()  # Skip Armed/Post frames

if merge_queue_depth > 100:
    identity_resolver.increase_batch_frequency()  # Run more often (every 15s)
```

**Why:** Prevents cascading failures, keeps system operational during load spikes.

---

## Performance Profiling

**Per-component timing:**
- ChunkProcessor: YOLO inference time
- TemporalLinker: Matching time
- IdentityResolver: Merge decision time
- EvidenceProcessor: Evidence processing time

**Per-camera performance:**
- Average processing time
- Auto-merge success rate
- Frame persistence lag

**Identify bottlenecks and problem cameras** (poor lighting, high activity, network issues).

---

## System Lifecycle Monitoring

**Bootstrap Phase Indicators:**
- Grid learning progress (% of camera pairs with learned paths)
- Face library growth (# samples per agent)
- Auto-merge rate improvement

**Target:** 0% → 80%+ auto-merge rate.

**Production Phase Indicators:**
- Stable auto-merge rate (> 80%)
- Low review queue depth (< 20)
- Consistent processing lag (< 15s)

**Goal:** Maintain performance, detect degradation early.

---

## Startup Health Checks

Before the main application loop begins, the system must pass a series of startup health checks to ensure critical dependencies are available.

### Database Connection
The system is critically dependent on the database for all stateful operations.

**Protocol:**
1. On startup, the system will attempt to connect to the configured PostgreSQL database.
2. If the connection fails, it will log the error and enter a retry loop.
3. It will attempt to reconnect every 5 seconds for a total of 60 seconds.
4. If a connection cannot be established after 60 seconds, the system will log a fatal error and exit with a non-zero status code.

This prevents the system from running in a broken state where it cannot record or retrieve information.