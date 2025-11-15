# Level 3 - Evidence Processing (EvidenceProcessor)

## Purpose
Resolve uncertain tracks when evidence arrives.

---

## Input/Output

**Input:** Evidence (face match, LLM decision, human review) + LocalTracks with identity_candidates

**Output:** Updated global_agent_id

**Timing:** Async (event-driven)

---

## Types of Evidence

### 1. Face Match Results

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

### 2. LLM Decisions

```python
{
    "track_id": 12345,
    "decision": "merge",  # or "reject" or "uncertain"
    "agent_id": 123,
    "confidence": "high",
    "reasoning": "Same clothing, body shape, similar gait"
}
```

**Decision:**
- If decision == "merge" and confidence == "high" → resolve
- If decision == "reject" → remove from candidates
- If decision == "uncertain" → leave in candidates

### 3. Human Reviews

```python
{
    "track_id": 12345,
    "decision": "merge",  # or "reject" or "new_agent"
    "agent_id": 123
}
```

**Decision:** Always trusted (human is ground truth)

---

## Evidence Processing Flow

### 1. Receive Evidence

```python
def process_evidence(evidence):
    track = db.get_track(evidence.track_id)
    if not track.identity_candidates:
        return  # Already resolved
    apply_evidence(track, evidence)
```

### 2. Filter identity_candidates

```python
if evidence.decision == "merge":
    track.global_agent_id = evidence.agent_id
    track.identity_candidates.clear()

elif evidence.decision == "reject":
    track.identity_candidates.discard(evidence.agent_id)
    if len(track.identity_candidates) == 1:
        track.global_agent_id = list(track.identity_candidates)[0]
```

### 3. Propagate Through Relationships

```python
def propagate_resolution(track):
    # Forward in time (same camera)
    if track.next_track:
        next_track = db.get_track(track.next_track)
        if not next_track.global_agent_id:
            next_track.global_agent_id = track.global_agent_id
            propagate_resolution(next_track)  # Recursive

    # Cross-camera merged tracks
    for merged_track_ref in track.merged_tracks:
        merged_track = db.get_track(merged_track_ref)
        if not merged_track.global_agent_id:
            merged_track.global_agent_id = track.global_agent_id
            propagate_resolution(merged_track)
```

### 4. Apply Process of Elimination

```python
def apply_elimination(resolved_track):
    """Agent_A confirmed in track_1 → remove Agent_A from all other uncertain tracks"""
    agent_id = resolved_track.global_agent_id
    timestamp = resolved_track.start_time

    # Find all uncertain tracks at same time
    uncertain_tracks = db.query(LocalTrack).filter(
        overlaps_time(timestamp),
        global_agent_id == None,
        identity_candidates.contains(agent_id)
    ).all()

    for track in uncertain_tracks:
        track.identity_candidates.discard(agent_id)
        if len(track.identity_candidates) == 1:
            track.global_agent_id = list(track.identity_candidates)[0]
            propagate_resolution(track)
            apply_elimination(track)  # Cascade
```

**Result:** Resolving one track can cascade through entire timeline.

---

## Review Queue Management

### Adding to Queue

```python
def queue_for_review(track, reason):
    review_queue.add({
        "track_id": track.id,
        "identity_candidates": track.identity_candidates,
        "reason": reason,
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

### Review Panel Generation

```python
{
    "track": track,
    "candidates": [
        {
            "agent_id": 123,
            "name": "Agent A",
            "recent_images": [img1, img2, img3],
            "last_seen": "Camera_B at T=10:34:21"
        }
    ],
    "context": {
        "previous_track": prev_track,
        "grid_transition": "possible (4.2s travel)",
        "alibi_check": "2 alternative sources"
    }
}
```

---

## Learning from Corrections

### Face Library Updates

```python
def on_human_validation(track, agent_id):
    track.global_agent_id = agent_id
    track.identity_candidates.clear()
    face_library.add_training_sample(track, agent_id, validated=True)
    propagate_resolution(track)
    apply_elimination(track)
```

### Grid Learning Updates

```python
def on_validated_merge(track_A, track_B):
    travel_time = track_B.start_time - track_A.end_time
    if is_clean_data(track_A, track_B):  # Solo tracks, high confidence
        grid.update(camera_A, camera_B, track_A.exit_cell, track_B.entry_cell, travel_time)
```

---

## Handling Uncertainty Persistence

**When uncertainty can't be resolved:**

```python
# Track remains uncertain
track.global_agent_id = None
track.identity_candidates = {Agent_A, Agent_B}

# Timeline shows all possibilities
timeline_A = db.query(LocalTrack).filter(
    global_agent_id == Agent_A OR identity_candidates.contains(Agent_A)
).all()
```

**Display:** "Agent A or Agent B (uncertain)"
