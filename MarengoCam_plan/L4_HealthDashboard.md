# Level 4 - Health Dashboard (System Monitoring UI)

## Purpose
Detailed specification for the system health monitoring dashboard that displays real-time performance metrics, alerts, and trends. Enables proactive issue detection and system tuning.

**Referenced from:** [L3_SystemHealth.md](L3_SystemHealth.md), [L3_Gui.md](L3_Gui.md)

**Implemented in:** L14_HealthDashboard.py

---

## Dashboard Layout

### Overall Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│ MarengoCam System Health                        [Last update: 2s]  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ ┌─────────────────────────┐  ┌─────────────────────────────────────┐
│ │ PROCESSING STATUS       │  │ QUEUE STATUS                        │
│ │                         │  │                                     │
│ │ Processing Lag:    8.2s │  │ Merge Queue:        42 tracks      │
│ │ GPU Utilization:    82% │  │ Review Queue:       18 tracks      │
│ │ Active Cameras:    3/10 │  │ Oldest Unresolved: 12 min          │
│ │                         │  │                                     │
│ │ [Lag Graph]             │  │ [Queue Depth Graph]                │
│ └─────────────────────────┘  └─────────────────────────────────────┘
│                                                                      │
│ ┌─────────────────────────┐  ┌─────────────────────────────────────┐
│ │ PERFORMANCE METRICS     │  │ ALERTS (2 active)                   │
│ │                         │  │                                     │
│ │ Auto-merge Rate:    84% │  │ ⚠️ GPU saturation (95%)             │
│ │ Frame Persist Lag: 3.1s │  │ ⚠️ Review queue high (> 20)         │
│ │ DB Query Avg:     48ms  │  │                                     │
│ │                         │  │ Recent (last hour):                 │
│ │ [Performance Trends]    │  │ • Processing lag spike (12:34)     │
│ └─────────────────────────┘  └─────────────────────────────────────┘
│                                                                      │
│ ┌──────────────────────────────────────────────────────────────────┐
│ │ PER-CAMERA PERFORMANCE                                           │
│ │                                                                  │
│ │ Camera       State  Lag    GPU%  Auto-merge  Frames/s  Issues   │
│ │ ──────────────────────────────────────────────────────────────  │
│ │ Driveway     Active 7.2s   28%   89%         3.1       -        │
│ │ FrontDoor    Active 9.1s   32%   82%         2.9       ⚠️ Lag   │
│ │ Backyard     Armed  12.3s  18%   -           1.2       -        │
│ │ Garage       Standby -     2%    -           0.1       -        │
│ │ ...                                                              │
│ └──────────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────────────┘
```

---

## Section 1: Processing Status

### Metrics Displayed

**Processing Lag (Primary Metric)**
- **Current value:** 8.2s
- **Target:** < 15s (green)
- **Warning:** 15-30s (yellow)
- **Critical:** > 30s (red)
- **Definition:** Time delay between video acquisition and LocalTrack creation

**Display format:**
```
Processing Lag: 8.2s  [████████░░] 55% of target
```

**Per-camera breakdown (expandable):**
- Show lag for each Active camera
- Identify slowest camera
- Trend: ↑ increasing, ↓ decreasing, → stable

---

**GPU Utilization**
- **Current value:** 82%
- **Target:** 70-90% (green)
- **Warning:** > 95% (saturated, red) or < 50% (underutilized, yellow)
- **Definition:** Percentage of GPU capacity used for YOLO inference

**Display format:**
```
GPU Utilization: 82%  [████████▓▓] Optimal
```

**Breakdown (expandable):**
- GPU load by camera state (Active, Armed, Post, Standby)
- Inference queue depth
- Average inference time per frame

---

**Active Cameras**
- **Current value:** 3/10
- **Definition:** Cameras in Active state (full inference + frame persistence)
- **Target:** Based on GPU capacity (typically 3-5 simultaneous)

**Display format:**
```
Active Cameras: 3/10
  • Driveway (since 10:32:15)
  • FrontDoor (since 10:34:01)
  • Backyard (since 10:35:42)
```

---

**Real-Time Graph (last 15 minutes)**
```
Lag (s)
30 ┤
25 ┤
20 ┤
15 ┤- - - - - - - - - Target - - - - - - - -
10 ┤     ╭─╮
 5 ┤─────╯ ╰─────────────────────────────────
 0 ┴─────────────────────────────────────────
   10:20    10:25    10:30    10:35    10:40
```

---

## Section 2: Queue Status

### Metrics Displayed

**Merge Queue Depth**
- **Current value:** 42 tracks
- **Target:** < 50 (green)
- **Warning:** 50-100 (yellow)
- **Critical:** > 100 (red)
- **Definition:** Number of uncertain tracks waiting for identity resolution

**Display format:**
```
Merge Queue: 42 tracks  [████░░░░░░] 84% of target
```

---

**Review Queue Depth**
- **Current value:** 18 tracks
- **Target:** < 20 (green)
- **Warning:** 20-50 (yellow)
- **Critical:** > 50 (red)
- **Definition:** Number of tracks queued for human/LLM review

**Display format:**
```
Review Queue: 18 tracks  [█████████░] 90% of target
  • High priority: 4
  • Medium priority: 10
  • Low priority: 4

[View Queue] button → opens L4_ReviewQueue
```

---

**Oldest Unresolved Track**
- **Current value:** 12 minutes
- **Target:** < 30 minutes
- **Warning:** > 2 hours
- **Definition:** Time since oldest uncertain track was created

**Display format:**
```
Oldest Unresolved: 12 min
  Track #12234 (Driveway, 2 candidates)
  [Jump to Review] button
```

---

**Queue Depth Graph (last 24 hours)**
```
Tracks
100 ┤                         ╭╮
 80 ┤                      ╭──╯╰─╮
 60 ┤                   ╭──╯     ╰╮
 40 ┤              ╭────╯         ╰───╮
 20 ┤─────────────╯                   ╰──────
  0 ┴────────────────────────────────────────
    12am  4am   8am   12pm  4pm   8pm   12am

Legend: ━ Merge Queue  ╍ Review Queue
```

---

## Section 3: Performance Metrics

### Metrics Displayed

**Auto-Merge Rate**
- **Current value:** 84% (last hour)
- **Target:** > 80% (after bootstrap)
- **Warning:** < 60%
- **Info:** > 90% (excellent)
- **Definition:** Percentage of tracks merged automatically (without human/LLM review)

**Display format:**
```
Auto-merge Rate: 84%  [████████▓░] Good
  • Last hour: 84% (42/50 merges)
  • Last 24h: 79% (312/395 merges)
  • Bootstrap progress: Day 12 (target reached)
```

**Trend indicator:**
- ↑ 5% from yesterday (improving)
- System learning indicator

---

**Frame Persistence Lag**
- **Current value:** 3.1s
- **Target:** < 5s (green)
- **Warning:** 5-10s (yellow)
- **Critical:** > 10s (red)
- **Definition:** Delay between frame acquisition and frame saved to disk

---

**Database Query Performance**
- **Current value:** 48ms average
- **Target:** < 100ms (green)
- **Warning:** 100-500ms (yellow)
- **Critical:** > 500ms (red)
- **Definition:** Average time for common queries

**Breakdown (expandable):**
- `get_agent_timeline`: 65ms
- `find_tracks`: 42ms
- `get_uncertain_tracks`: 31ms

---

**Performance Trends (last 7 days)**
```
Auto-merge Rate (%)
100 ┤                           ╭───────────
 80 ┤             ╭────────────╯
 60 ┤        ╭────╯
 40 ┤───────╯                    Bootstrap
 20 ┤                                 │
  0 ┴──────────────────────────────────────
    Day 5   Day 7   Day 9   Day 11   Day 13
```

---

## Section 4: Alerts

### Alert Types

**Critical Alerts (Red 🔴)**
- Processing lag > 30s
- GPU saturated (> 95%)
- Review queue depth > 50
- Database query time > 500ms
- Frame persistence lag > 10s

**Warning Alerts (Yellow ⚠️)**
- Processing lag > 15s
- GPU underutilized (< 50%)
- Review queue depth > 20
- Merge queue depth > 50
- Auto-merge rate < 60%

**Info Alerts (Blue ℹ️)**
- Auto-merge rate > 90% (excellent!)
- System learning milestone (e.g., "Grid 80% learned")

### Alert Display

```
┌──────────────────────────────────────────┐
│ ALERTS (2 active)                         │
│                                           │
│ 🔴 CRITICAL: GPU saturation (95%)        │
│    Since: 10:38:12 (3 min ago)           │
│    Impact: Frame drops likely            │
│    Action: [Reduce sampling rate]        │
│                                           │
│ ⚠️ WARNING: Review queue high (21)       │
│    Since: 09:15:00 (1.5 hours ago)       │
│    Impact: Identity resolution delayed   │
│    Action: [Open Review Queue]           │
│                                           │
│ Recent (last hour):                       │
│ • 10:34 - Processing lag spike (18.2s)   │
│   Resolved: Auto-remediation reduced lag │
│ • 09:52 - Merge queue depth warning      │
│   Resolved: Evidence processing caught up│
└──────────────────────────────────────────┘
```

### Alert Actions

**Auto-remediation (if enabled):**
- GPU saturated → reduce sampling rate for Armed/Post cameras
- Processing lag high → prioritize Active cameras
- Merge queue deep → increase IdentityResolver batch frequency

**Manual actions:**
- [Acknowledge] - Dismiss alert (logged)
- [Investigate] - Open detailed diagnostics
- [Remediate] - Trigger specific fix

---

## Section 5: Per-Camera Performance

### Table Columns

**Camera:** Camera ID
**State:** Standby / Armed / Active / Post
**Lag:** Processing lag for this camera (if Active)
**GPU%:** GPU allocation percentage
**Auto-merge:** Auto-merge rate for tracks from this camera
**Frames/s:** Current inference rate
**Issues:** Any camera-specific warnings

### Example Row

```
Camera      State   Lag    GPU%  Auto-merge  Frames/s  Issues
──────────────────────────────────────────────────────────────
FrontDoor   Active  9.1s   32%   82%         2.9       ⚠️ Lag
```

**Expandable details (click row):**
```
FrontDoor - Active since 10:34:01 (6.5 min)
  • Current track: Track #12456 (duration 3.2s)
  • Processing lag: 9.1s (above target)
  • Auto-merge rate: 82% (last 10 tracks: 8/10)
  • Issue: High activity (3 concurrent agents)
  • Recommendation: Normal operation (lag expected during high activity)
```

### Sorting & Filtering

**Sort by:**
- State (Active first)
- Lag (highest first)
- Auto-merge rate (lowest first)
- Issues (warnings first)

**Filter by:**
- State (show only Active)
- Issues (show only cameras with warnings)

---

## Dashboard Interactions

### Refresh Rate

**Auto-refresh:** Every 5 seconds (configurable)

**Manual refresh:** [Refresh Now] button

**Real-time updates:** WebSocket for alerts (no delay)

### Navigation

**Drill-down:**
- Click alert → detailed diagnostics
- Click camera → camera-specific dashboard
- Click queue metric → open review queue
- Click graph → historical trends (last 7 days)

**Quick actions:**
- [Pause auto-refresh] - Stop updates
- [Export metrics] - Download CSV
- [Email report] - Send summary to admin

---

## Mobile/Compact View

**Simplified layout for mobile:**
```
┌───────────────────────────┐
│ System Health             │
├───────────────────────────┤
│ Processing Lag: 8.2s ✓    │
│ GPU: 82% ✓                │
│ Review Queue: 18 ⚠️       │
│                           │
│ [Alerts: 2 active] >      │
│ [Per-Camera] >            │
│ [Historical] >            │
└───────────────────────────┘
```

---

## Related Documents

### Tactical Context
- **L3_SystemHealth.md** - Metrics definitions, thresholds, alerting logic
- **L3_Gui.md** - Overall GUI structure

### Other L4 Concepts
- **L4_ReviewQueue.md** - Review queue (linked from dashboard)

### Implementation
- **L14_HealthDashboard.py** - Dashboard rendering, real-time updates
- **L14_MetricsCollector.py** - Metrics computation, storage

### Algorithms
- **L12_AlertingLogic.md** - Alert threshold computation, auto-remediation triggers
