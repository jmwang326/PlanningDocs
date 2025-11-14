# Level 4 - Review Queue (Priority Management)

## Purpose
Detailed specification for the review queue that manages uncertain tracks awaiting human/LLM adjudication. Defines prioritization logic, queue display, and workflow.

**Referenced from:** [L3_EvidenceProcessing.md](L3_EvidenceProcessing.md)

**Implemented in:** L14_ReviewQueue.py

---

## Queue Entry Structure

### Data Model

```python
ReviewQueueEntry {
    track_id: int,
    camera_id: str,
    timestamp: float,
    duration: float,
    identity_candidates: set[int],  # Set of candidate agent IDs
    reason: str,  # Why queued for review
    priority: float,  # Computed priority score
    evidence_summary: dict,  # Pre-computed evidence
    queued_at: float,  # When added to queue
    thumbnail: str  # Path to representative frame
}
```

### Queue Reasons

**Why track is queued for review:**
- `"ambiguous_timing"` - Multiple candidates fit grid transition
- `"multiple_candidates"` - Several plausible identities
- `"portal_transition"` - Crossed portal with uncertain identity
- `"weak_evidence"` - No strong face/grid match
- `"concurrent_activity"` - Multiple agents visible simultaneously
- `"face_below_threshold"` - Face match 0.60-0.74 (uncertain range)
- `"alibi_ambiguous"` - Alternative sources exist

---

## Priority Calculation

### Priority Formula

```python
priority = (recency_score * 0.4) +
           (simplicity_score * 0.3) +
           (impact_score * 0.3)
```

### Component Scores

**1. Recency Score (0.4 weight)**

**Purpose:** Prioritize recent tracks (fresher evidence, easier to review)

```python
recency_score = 1.0 / (now - track.queued_at)
```

**Normalization:** Scale to 0.0-1.0 range
- Track < 5 minutes old: score = 1.0
- Track 1 hour old: score = 0.5
- Track > 24 hours old: score = 0.1

---

**2. Simplicity Score (0.3 weight)**

**Purpose:** Prioritize tracks with fewer candidates (easier decisions)

```python
simplicity_score = 1.0 / len(track.identity_candidates)
```

**Examples:**
- 2 candidates: score = 0.5
- 3 candidates: score = 0.33
- 5 candidates: score = 0.2

**Rationale:** Binary decisions (Agent A vs Agent B) are easier than multi-way choices.

---

**3. Impact Score (0.3 weight)**

**Purpose:** Prioritize tracks that block other resolutions (cascading impact)

```python
impact_score = count_blocked_tracks(track)
```

**Blocked tracks definition:**
- Tracks with `previous_track` or `next_track` pointing to uncertain track
- Tracks in `merged_tracks` relationships
- Tracks with overlapping `identity_candidates` (process of elimination)

**Normalization:**
- 0 blocked tracks: score = 0.1 (isolated, low impact)
- 1-3 blocked: score = 0.5
- 5+ blocked: score = 1.0 (high cascading impact)

---

### Priority Thresholds

**High Priority (score > 0.7):**
- Show at top of queue
- Highlight in red
- Send email notification (if configured)

**Medium Priority (0.4 < score < 0.7):**
- Standard queue position
- Default display

**Low Priority (score < 0.4):**
- Show at bottom
- Can be deferred or auto-assigned to LLM

---

## Queue Display

### List View

```
┌───────────────────────────────────────────────────────────────────┐
│ REVIEW QUEUE (23 pending)                    [Sort: Priority ▼]  │
├───────────────────────────────────────────────────────────────────┤
│                                                                    │
│ 🔴 HIGH PRIORITY (4)                                              │
│                                                                    │
│ ┌─────┬──────────────────────────────────────────┬──────────────┐│
│ │[img]│ Track #12345 - Driveway                 │ Priority: 0.82││
│ │     │ 2 candidates: Agent A, Agent B          │ 3 min ago    ││
│ │     │ Reason: ambiguous_timing                │ [Review]     ││
│ ├─────┼──────────────────────────────────────────┼──────────────┤│
│ │[img]│ Track #12398 - FrontDoor                │ Priority: 0.78││
│ │     │ 3 candidates: Agent C, Agent D, new     │ 8 min ago    ││
│ │     │ Reason: multiple_candidates             │ [Review]     ││
│ └─────┴──────────────────────────────────────────┴──────────────┘│
│                                                                    │
│ 🟡 MEDIUM PRIORITY (12)                                           │
│ ...                                                                │
│                                                                    │
│ ⚪ LOW PRIORITY (7)                                               │
│ ...                                                                │
└───────────────────────────────────────────────────────────────────┘
```

### Card View (Detailed)

**Each queue entry shows:**
- **Thumbnail:** Best frame from track (face visible if available)
- **Track metadata:** Camera, timestamp, duration
- **Candidates:** Agent IDs + names (if assigned)
- **Reason:** Why queued
- **Priority score:** 0.0-1.0
- **Age:** Time since queued
- **Action button:** "Review" → Opens L4_ReviewPanel

---

## Queue Operations

### Add to Queue

**Triggered by:** IdentityResolver marks track as `QUEUE_FOR_REVIEW`

```python
def add_to_queue(track, reason):
    entry = ReviewQueueEntry(
        track_id=track.id,
        camera_id=track.camera_id,
        timestamp=track.start_time,
        duration=track.end_time - track.start_time,
        identity_candidates=track.identity_candidates.copy(),
        reason=reason,
        priority=calculate_priority(track),
        queued_at=now()
    )

    review_queue.add(entry)

    # Pre-generate review panel for fast access
    panel_assembly_service.prepare_panel(entry)
```

### Remove from Queue

**Triggered when:**
- Human/LLM makes decision → resolved
- Evidence arrives (face match, etc.) → auto-resolved
- Process of elimination → narrowed to single candidate
- Track deleted/invalidated

```python
def remove_from_queue(track_id):
    review_queue.remove(track_id)
    panel_assembly_service.cleanup(track_id)
```

### Update Priority

**Re-compute priority when:**
- New evidence narrows candidates (simplicity changes)
- More tracks become blocked (impact changes)
- Time passes (recency changes)

```python
def update_priorities():
    for entry in review_queue:
        entry.priority = calculate_priority(entry.track)

    review_queue.sort(key=lambda e: e.priority, reverse=True)
```

**Frequency:** Every 60 seconds (background task)

---

## Queue Workflow

### 1. Queueing Phase

**IdentityResolver** evaluates track and decides:
- Auto-merge (high confidence) → no queue
- Reject (impossible) → no queue
- Uncertain (ambiguous) → add to queue

### 2. Review Phase

**Human/LLM** opens queue, sees prioritized list, selects entry:
- Opens L4_ReviewPanel
- Makes decision (merge/reject/skip)
- Decision processed by EvidenceProcessor

### 3. Resolution Phase

**EvidenceProcessor** applies decision:
- Updates `track.global_agent_id`
- Propagates through relationships
- Triggers process of elimination
- Removes from queue

### 4. Cascading Phase

**Process of elimination** may resolve other tracks:
- Narrowed candidates → auto-resolve
- Updated priority for remaining queue entries
- May remove additional tracks from queue

---

## Queue Strategies

### Human Review Strategy

**Best for:**
- High-priority tracks (score > 0.7)
- Complex decisions (> 3 candidates)
- First 50-100 tracks (bootstrap phase)

**Workflow:**
- Human reviews queue in priority order
- Aims for 10-20 reviews per session
- Skips low-priority or defers complex cases to LLM

### LLM Review Strategy

**Best for:**
- Medium-priority tracks (0.4 < score < 0.7)
- Binary decisions (2 candidates)
- After bootstrap phase (grid learned, face library trained)

**Workflow:**
- LLM batch-processes queue (10 tracks at a time)
- High-confidence LLM decisions applied automatically
- Low-confidence LLM decisions escalated to human

### Hybrid Strategy (Recommended)

**Phase 1 (Bootstrap):**
- Human reviews all high-priority (train system)
- LLM reviews low-priority (filter noise)

**Phase 2 (Production):**
- LLM reviews 80% of queue (binary decisions)
- Human reviews 20% (complex cases, LLM escalations)

---

## Queue Metrics

### Performance Indicators

**Queue depth:**
- Target: < 20 tracks
- Warning: > 50 tracks
- Critical: > 100 tracks

**Average time-in-queue:**
- Target: < 30 minutes
- Warning: > 2 hours
- Critical: > 24 hours

**Resolution rate:**
- Target: > 10 tracks/hour (human)
- Target: > 50 tracks/hour (LLM)

**Auto-resolution rate:**
- Target: > 80% (after bootstrap)
- Indicator of system learning

---

## Queue Filtering & Search

### Filter Options

**By camera:**
- Show only tracks from specific camera
- Useful for debugging camera-specific issues

**By candidate:**
- Show only tracks with specific agent as candidate
- Useful for focused identity resolution

**By reason:**
- Group by queue reason
- Identify systematic issues

**By priority:**
- Show only high-priority
- Focus on urgent reviews

### Search

**Text search:**
- Agent names
- Camera IDs
- Track IDs

**Time range:**
- Last hour / Last day / Custom range
- Useful for session-based review

---

## Related Documents

### Tactical Context
- **L3_EvidenceProcessing.md** - Review queue management logic
- **L3_IdentityResolution.md** - When tracks are queued
- **L3_SystemHealth.md** - Queue depth monitoring

### Other L4 Concepts
- **L4_ReviewPanel.md** - What reviewers see when they open a queue entry

### Implementation
- **L14_ReviewQueue.py** - Queue data structure, priority calculation
- **L14_PanelAssemblyService.py** - Pre-generation of review panels

### Algorithms
- **L12_ProcessOfElimination.md** - Impact score calculation (blocked tracks)
