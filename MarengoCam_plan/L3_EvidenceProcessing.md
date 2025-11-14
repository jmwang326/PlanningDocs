# Level 3 - Evidence Processing (EvidenceProcessor)

## Purpose
Tactical guide for resolving uncertain tracks when evidence arrives. This handler processes face match results, LLM decisions, and human reviews to update identity_candidates and resolve ambiguity.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for EvidenceProcessor architecture overview.

---

## Input/Output

**Input:**
- Evidence (face match result, LLM decision, human review)
- LocalTracks with identity_candidates (uncertain tracks)

**Output:** Updated global_agent_id on affected LocalTracks

**Timing:** Async (event-driven, triggered when evidence arrives)

---

## Types of Evidence

### 1. Face Match Results (from CodeProject.AI)

**Arrives:** Asynchronously after face crops submitted

**Format:**
```python
{
    "track_id": 12345,
    "agent_id": 123,
    "confidence": 0.82
}
```

**Decision:**
- If confidence ≥ 0.75 → resolve to agent_id
- If confidence < 0.75 → add to identity_candidates (weak evidence)

**See:** L13_FaceRecognition.md for CodeProject.AI integration

---

### 2. LLM Decisions

**Arrives:** After LLM reviews visual comparison panels

**Format:**
```python
{
    "track_id": 12345,
    "decision": "merge",  # or "reject" or "uncertain"
    "agent_id": 123,
    "confidence": "high",  # or "medium" or "low"
    "reasoning": "Same clothing, body shape, similar gait"
}
```

**Decision:**
- If decision == "merge" and confidence == "high" → resolve to agent_id
- If decision == "reject" → remove from identity_candidates
- If decision == "uncertain" → leave in identity_candidates

**See:** L3_Gui.md for review panel generation

---

### 3. Human Reviews

**Arrives:** After human reviews track in GUI

**Format:**
```python
{
    "track_id": 12345,
    "decision": "merge",  # or "reject" or "new_agent"
    "agent_id": 123
}
```

**Decision:** Always trusted (human is ground truth)
- If decision == "merge" → resolve to agent_id
- If decision == "reject" → remove from identity_candidates
- If decision == "new_agent" → create new global_agent_id

**See:** L3_Gui.md for review queue UI

---

## Evidence Processing Flow

### 1. Receive Evidence

```python
def process_evidence(evidence):
    track = db.get_track(evidence.track_id)

    if not track.identity_candidates:
        # Track already resolved, ignore
        return

    # Apply evidence
    apply_evidence(track, evidence)
```

---

### 2. Filter identity_candidates

```python
def apply_evidence(track, evidence):
    if evidence.decision == "merge":
        # Resolve to single agent
        track.global_agent_id = evidence.agent_id
        track.identity_candidates.clear()

    elif evidence.decision == "reject":
        # Remove rejected agent from candidates
        track.identity_candidates.discard(evidence.agent_id)

        # If only one candidate remains, resolve
        if len(track.identity_candidates) == 1:
            track.global_agent_id = list(track.identity_candidates)[0]
            track.identity_candidates.clear()
```

---

### 3. Propagate Through Relationships

```python
def propagate_resolution(track):
    """
    When track resolved, propagate through:
    - next_track (same camera, future chunks)
    - merged_tracks (other cameras, same agent)
    """
    # Propagate forward in time (same camera)
    if track.next_track:
        next_track = db.get_track(track.next_track)
        if not next_track.global_agent_id:
            next_track.global_agent_id = track.global_agent_id
            propagate_resolution(next_track)  # Recursive

    # Propagate to merged tracks (cross-camera)
    for merged_track_ref in track.merged_tracks:
        merged_track = db.get_track(merged_track_ref)
        if not merged_track.global_agent_id:
            merged_track.global_agent_id = track.global_agent_id
            propagate_resolution(merged_track)  # Recursive
```

**Why recursive:** One resolution can cascade through entire timeline.

---

### 4. Apply Process of Elimination

```python
def apply_elimination(resolved_track):
    """
    When Agent_A confirmed in track_1, remove Agent_A from
    all other uncertain tracks (can't be in two places at once)
    """
    agent_id = resolved_track.global_agent_id
    timestamp = resolved_track.start_time

    # Find all uncertain tracks at same time
    uncertain_tracks = db.query(LocalTrack).filter(
        start_time <= timestamp,
        end_time >= timestamp,
        global_agent_id == None,
        identity_candidates.contains(agent_id)
    ).all()

    for track in uncertain_tracks:
        track.identity_candidates.discard(agent_id)

        # If only one candidate remains, resolve
        if len(track.identity_candidates) == 1:
            track.global_agent_id = list(track.identity_candidates)[0]
            track.identity_candidates.clear()
            propagate_resolution(track)
            apply_elimination(track)  # Cascade
```

**Result:** Resolving one track can trigger cascade of resolutions.

**See:** L12_ProcessOfElimination.md for cascading logic details

---

## Review Queue Management

### Adding to Review Queue

**Triggers:** IdentityResolver marks track as "QUEUE_FOR_REVIEW"

```python
def queue_for_review(track, reason):
    review_queue.add({
        "track_id": track.id,
        "identity_candidates": track.identity_candidates,
        "reason": reason,  # "ambiguous_timing", "multiple_candidates", etc.
        "priority": calculate_priority(track)
    })
```

### Priority Calculation

**Higher priority:**
- More recent tracks (fresh data)
- Fewer candidates (easier to resolve)
- More subsequent tracks blocked (cascading impact)

```python
def calculate_priority(track):
    recency = 1.0 / (now - track.end_time)
    simplicity = 1.0 / len(track.identity_candidates)
    impact = count_blocked_tracks(track)

    return recency + simplicity + impact
```

---

### Review Panel Generation

**For each uncertain track, generate comparison panel:**

```python
{
    "track": track,
    "candidates": [
        {
            "agent_id": 123,
            "name": "Agent A",
            "recent_images": [img1, img2, img3],
            "last_seen": "Camera_B at T=10:34:21"
        },
        {
            "agent_id": 456,
            "name": "Agent B",
            "recent_images": [img4, img5, img6],
            "last_seen": "Camera_C at T=10:33:45"
        }
    ],
    "context": {
        "previous_track": prev_track,
        "grid_transition": "possible (4.2s travel)",
        "alibi_check": "2 alternative sources"
    }
}
```

**See:** L3_Gui.md for review panel UI

---

## Learning from Corrections

### Face Library Updates

**When human validates merge:**

```python
def on_human_validation(track, agent_id):
    # Resolve track
    track.global_agent_id = agent_id
    track.identity_candidates.clear()

    # Add to face library (validated ground truth)
    face_library.add_training_sample(track, agent_id, validated=True)

    # Propagate
    propagate_resolution(track)
    apply_elimination(track)
```

**Result:** Face library improves from validated merges only (no pollution from uncertain data).

**See:** L13_FaceRecognition.md for library management

---

### Grid Learning Updates

**When validated merge confirms transition:**

```python
def on_validated_merge(track_A, track_B):
    # Update grid with validated travel time
    travel_time = track_B.start_time - track_A.end_time

    if is_clean_data(track_A, track_B):
        # Solo tracks, high confidence, no ambiguity
        grid.update(
            camera_A, camera_B,
            track_A.exit_cell, track_B.entry_cell,
            travel_time
        )
```

**Conditions for grid learning:**
- Solo tracks (max_concurrent_agents = 1)
- High-confidence merge
- No ambiguity

**See:** L12_GridLearning.md for update algorithm

---

## Handling Uncertainty Persistence

### When Uncertainty Can't Be Resolved

**Scenarios:**
- Face occluded in all frames
- Multiple candidates with similar appearance
- No strong evidence arrives

**Strategy:** Accept uncertainty, operate with partial timeline

```python
# Track remains uncertain
track.global_agent_id = None
track.identity_candidates = {Agent_A, Agent_B}

# Timeline shows all possibilities
timeline_A = db.query(LocalTrack).filter(
    global_agent_id == Agent_A OR identity_candidates.contains(Agent_A)
).all()
```

**Display in GUI:** "Agent A or Agent B (uncertain)"

**See:** L3_Gui.md for uncertainty visualization

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - EvidenceProcessor overview
- **L2_Strategy.md** - Accept uncertainty, no probability scores

### Previous Handler
- **L3_IdentityResolution.md** - Creates uncertain tracks (identity_candidates)

### Implementation
- **L12_ProcessOfElimination.md** - Cascading elimination logic
- **L13_FaceRecognition.md** - CodeProject.AI integration, face library
- **L3_Gui.md** - Review queue, comparison panels
