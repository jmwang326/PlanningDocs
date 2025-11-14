# Level 3 - System Health (System Health Monitor)

## Purpose
Tactical guide for monitoring system performance, detecting issues, and alerting on problems. This ensures MarengoCam runs smoothly and processes video in real-time.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for SystemHealth architecture overview.

---

## Metrics to Monitor

### 1. Processing Lag

**What:** Time delay between video acquisition and LocalTrack creation

**Target:** < 15s (chunk processing should complete within 5s of chunk end)

**Measurement:**
```python
lag = now - (chunk_end_time + processing_time)
```

**Alert thresholds:**
- Warning: lag > 15s
- Critical: lag > 30s

**Causes:**
- GPU overloaded
- Too many cameras active simultaneously
- Inference Manager not prioritizing correctly

**See:** L13_InferenceManager.md for GPU scheduling

---

### 2. GPU Utilization

**What:** Percentage of GPU capacity used for YOLO inference

**Target:** 70-90% (high utilization without saturation)

**Measurement:**
```python
gpu_util = current_inference_load / max_gpu_capacity
```

**Alert thresholds:**
- Warning: gpu_util > 95% (saturated, may drop frames)
- Warning: gpu_util < 50% (underutilized, wasting capacity)

**Causes:**
- Too many Active cameras (> GPU capacity)
- Inference rate miscalibrated
- YOLO model too heavy for GPU

---

### 3. Merge Queue Depth

**What:** Number of uncertain tracks waiting for identity resolution

**Target:** < 50 tracks

**Measurement:**
```python
queue_depth = count(tracks where global_agent_id == None and identity_candidates.length > 0)
```

**Alert thresholds:**
- Warning: queue_depth > 50
- Critical: queue_depth > 100

**Causes:**
- Face recognition service slow/down
- Too much concurrent activity (many uncertain tracks)
- Grid learning incomplete (no learned paths)

---

### 4. Review Queue Depth

**What:** Number of tracks queued for human/LLM review

**Target:** < 20 tracks

**Measurement:**
```python
review_queue_depth = count(review_queue)
```

**Alert thresholds:**
- Warning: review_queue_depth > 20
- Critical: review_queue_depth > 50

**Causes:**
- High ambiguity rate (many uncertain merges)
- LLM service slow/down
- Human not reviewing regularly

---

### 5. Auto-Merge Rate

**What:** Percentage of tracks merged automatically (without human/LLM review)

**Target:** > 80% (after bootstrap phase)

**Measurement:**
```python
auto_merge_rate = count(auto_merged) / count(all_merges)
```

**Alert thresholds:**
- Warning: auto_merge_rate < 60%
- Info: auto_merge_rate > 90% (excellent!)

**Causes (low rate):**
- Face library incomplete (need more training)
- Grid learning incomplete (no learned paths)
- High concurrent activity (multiple people)

---

### 6. Frame Persistence Lag

**What:** Delay between frame acquisition and frame saved to disk

**Target:** < 5s

**Measurement:**
```python
persistence_lag = frame_save_time - frame_acquisition_time
```

**Alert thresholds:**
- Warning: persistence_lag > 5s
- Critical: persistence_lag > 10s

**Causes:**
- Disk I/O bottleneck
- Too many Active cameras writing simultaneously
- Network latency (if using network storage)

---

### 7. Database Query Performance

**What:** Average time for common queries (get_agent_timeline, find_tracks, etc.)

**Target:** < 100ms

**Measurement:**
```python
avg_query_time = sum(query_times) / count(queries)
```

**Alert thresholds:**
- Warning: avg_query_time > 100ms
- Critical: avg_query_time > 500ms

**Causes:**
- Missing database indexes
- Too many LocalTracks (need archival strategy)
- Query inefficiency

---

## Health Dashboard

### Real-Time Display

**Sections:**
1. **Processing Status**
   - Current lag (per camera)
   - GPU utilization
   - Active cameras count

2. **Queue Status**
   - Merge queue depth
   - Review queue depth
   - Oldest unresolved track

3. **Performance Metrics**
   - Auto-merge rate (last hour)
   - Frame persistence lag
   - Database query performance

4. **Alerts**
   - Warning-level issues
   - Critical issues
   - Recent alerts history

**See:** L4_HealthDashboard.md for dashboard specification

---

### Historical Trends

**Track over time:**
- Processing lag trends
- GPU utilization patterns
- Auto-merge rate improvement
- Queue depth fluctuations

**Why:** Identify gradual degradation, seasonal patterns, system learning progress.

**Visualization:**
- Time-series graphs (last 24 hours, last 7 days)
- Heatmaps (by camera, by time of day)

---

## Alerting

### Alert Levels

**Info:** Positive indicators (auto-merge rate > 90%)

**Warning:** Potential issues requiring attention (lag > 15s, queue depth > 50)

**Critical:** Immediate action required (lag > 30s, GPU saturated, service down)

---

### Alert Delivery

**Channels:**
- Dashboard (real-time display)
- Email (for critical alerts only)
- System log (all alerts)

**Throttling:** Don't spam on repeated alerts (wait 5 minutes between same alert)

**See:** L3_Configuration.md for alert configuration

---

## Automatic Remediation

### Self-Healing Actions

**When GPU saturated:**
```python
if gpu_util > 95%:
    inference_manager.reduce_sampling_rate()
    # Reduce Armed/Post camera sampling temporarily
```

**When processing lag high:**
```python
if lag > 30s:
    inference_manager.prioritize_active_cameras()
    # Skip some Armed/Post frames to catch up
```

**When merge queue too deep:**
```python
if merge_queue_depth > 100:
    identity_resolver.increase_batch_frequency()
    # Run IdentityResolver more often (every 15s instead of 30s)
```

**Why:** Prevents cascading failures, keeps system operational during load spikes.

---

## Performance Profiling

### Per-Component Timing

**Track time spent in each handler:**
- ChunkProcessor: YOLO inference time
- TemporalLinker: Matching time
- IdentityResolver: Merge decision time
- EvidenceProcessor: Evidence processing time

**Identify bottlenecks:**
```python
if chunk_processor_time > 5s:
    # YOLO inference slow (GPU issue? Model too heavy?)
```

**See:** L13_Handlers.md for profiling instrumentation

---

### Per-Camera Performance

**Track metrics per camera:**
- Average processing time
- Auto-merge success rate
- Frame persistence lag

**Identify problem cameras:**
- Poor lighting → low confidence detections
- High activity → GPU overload
- Network issues → frame acquisition lag

---

## System Lifecycle Monitoring

### Bootstrap Phase Indicators

**Metrics during initial learning:**
- Grid learning progress (% of camera pairs with learned paths)
- Face library growth (# of training samples per agent)
- Auto-merge rate improvement over time

**Goal:** Track system "learning" from 0% to 80%+ auto-merge rate.

**See:** L5_Bootstrap.md for bootstrap metrics

---

### Production Phase Indicators

**Metrics during normal operation:**
- Stable auto-merge rate (> 80%)
- Low review queue depth (< 20)
- Consistent processing lag (< 15s)

**Goal:** Maintain performance, detect degradation early.

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - SystemHealth overview

### Other L3 Components
- **L3_ChunkProcessing.md** - Processing lag source
- **L3_IdentityResolution.md** - Merge queue, auto-merge rate
- **L3_EvidenceProcessing.md** - Review queue
- **L3_Gui.md** - GUI overview
- **L3_Configuration.md** - Alert configuration

### L4 Concepts
- **L4_HealthDashboard.md** - System health dashboard specification

### Implementation
- **L13_InferenceManager.md** - GPU scheduling, adaptive sampling
- **L13_Handlers.md** - Performance profiling instrumentation
- **L5_Bootstrap.md** - Bootstrap phase metrics
