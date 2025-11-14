# Level 3 - Identity Resolution (IdentityResolver)

## Purpose
Tactical guide for assigning global_agent_id to LocalTracks (cross-camera merging). This is the core decision-making component that determines which tracks belong to the same person.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for IdentityResolver architecture overview.

---

## Input/Output

**Input:** LocalTrack without global_agent_id (after TemporalLinker)

**Output:** global_agent_id (certain) OR identity_candidates (uncertain)

**Timing:** Periodic batch processing (e.g., every 30s)

**Why batched:** Cross-camera merging needs to see recent tracks from all cameras. Batching is more efficient than per-track processing.

---

## Evidence Hierarchy

IdentityResolver applies evidence in strict order (strongest first):

### 1. Face Recognition (Definitive)

**Method:** CodeProject.AI face library

**Confidence threshold:** ≥ 0.75

**Decision:** AUTO-MERGE

```python
if face_match_confidence >= 0.75:
    track.global_agent_id = matched_agent_id
    return "AUTO_MERGE"
```

**Why definitive:** Face recognition is the most reliable evidence.

**See:** L13_FaceRecognition.md for CodeProject.AI integration

---

### 2. Grid Overlap (Definitive if Single Agent)

**Condition:** Travel time < 0.4s between cameras

**What it means:** Cameras see overlapping areas (same physical space)

**Decision logic:**
```python
travel_time = track_B.start_time - track_A.end_time

if travel_time < 0.4:  # Overlap zone
    if only_one_agent_visible:
        track_B.global_agent_id = track_A.global_agent_id
        return "AUTO_MERGE"
    else:
        # Multiple candidates in overlap zone
        track_B.identity_candidates = {Agent_A, Agent_B, ...}
        return "QUEUE_FOR_REVIEW"
```

**Key insight:** No time gap = same space = same agent (if only one visible)

**See:** L4_GridLearning.md for grid learning concept, overlap zone detection

---

### 3. Grid Transition + Alibi Check (Definitive if No Alternatives)

**Condition:** Travel time fits learned 6x6 grid pattern

**Decision logic:**
```python
travel_time = track_B.start_time - track_A.end_time
grid_time = grid.get_time(camera_A, camera_B, exit_cell_A, entry_cell_B)

if abs(travel_time - grid_time) < variance_threshold:
    # Plausible transition
    if alibi_check_passes(track_B, travel_time):
        # No other agents could have reached track_B
        track_B.global_agent_id = track_A.global_agent_id
        return "AUTO_MERGE"
    else:
        # Alternative sources exist
        track_B.identity_candidates = {Agent_A, Agent_X, ...}
        return "QUEUE_FOR_REVIEW"
```

**Alibi check:** "Could entry agent have come from elsewhere?"
- Check all other cameras: "Could any other recent track have arrived at this entry point at this time?"
- If NO → must be from track_A (process of elimination)
- If YES → ambiguous (multiple candidates)

**See:** L4_AlibiCheck.md for alibi check concept and process of elimination

---

### 4. LLM / Human Adjudication (Fallback)

**Condition:** Time/space plausible but ambiguous

**Decision:** QUEUE_FOR_REVIEW

```python
track.identity_candidates = {Agent_A, Agent_B, ...}
queue_for_review(track, reason="ambiguous_timing")
```

**Triggers:**
- Multiple candidates fit grid transition
- Moderate Re-ID similarity (not high enough for auto-merge)
- Portal transitions with multiple recent entries

**See:** L3_EvidenceProcessing.md for review queue handling

---

### 5. Reject (Definitive)

**Conditions:**
- Face mismatch (different person, high confidence)
- Spatial/temporal impossibility (travel time too short or too long)
- Alibi check fails (seen elsewhere simultaneously)

**Decision:** Do not merge

```python
if face_mismatch or impossible_timing:
    # Do not assign global_agent_id from this candidate
    # May try other candidates or create new agent
    return "REJECT"
```

---

## Grid-Based Spatial Learning

### What Is the 6x6 Grid

**Per camera:** 6x6 grid cells overlaid on camera FOV

**Per camera pair:** Learned minimum travel times between exit/entry cells

**Example:**
```python
grid[CameraA→CameraB][(exit_cell_5_2)→(entry_cell_0_3)] = 1.5s
```

**Meaning:** Fastest observed time for agent to travel from CameraA cell (5,2) to CameraB cell (0,3) is 1.5 seconds.

---

### How Grid Learning Works

**1. Observation:** Person walks from Camera A to Camera B (validated merge)

**2. Measurement:**
```python
travel_time = track_B.start_time - track_A.end_time
```

**3. Mapping:**
```python
exit_cell = track_A.exit_cell  # (5, 2)
entry_cell = track_B.entry_cell  # (0, 3)
```

**4. Learning:**
```python
current_time = grid[A→B][(5,2)→(0,3)]
if travel_time < current_time:
    grid[A→B][(5,2)→(0,3)] = travel_time  # Keep minimum
```

**5. Future merging:**
```python
if abs(new_travel_time - grid_time) < variance:
    # Plausible transition
```

---

### What Gets Learned (Clean Data Only)

**Include:**
- Solo tracks (max_concurrent_agents = 1)
- High-confidence merges (face match ≥ 0.75 OR validated human review)
- Sustained movement (≥ 3s duration)
- Clean observations (no occlusions, good lighting)

**Exclude:**
- Groups (multiple people together) - ambiguous which person traveled
- Ambiguous merges (uncertain identity)
- Sporadic vehicle detections (person in car)
- Low-quality tracks (partial visibility)

**Why:** Prevents noise pollution. Grid learns only from reliable data.

**See:** L4_GridLearning.md for grid learning concept and bootstrap strategy

---

### Overlap Zones (Fastest Merging)

**Condition:** Travel time < 0.4s

**What it means:** Cameras see overlapping physical space

**Grid cell marking:**
```python
grid[A→B][(5,2)→(0,3)] = 0  # Overlap zone
```

**Future merging:**
```python
if grid_time == 0 and only_one_agent_visible:
    # Instant auto-merge (must be same agent)
```

**Key insight:** No time gap = same space = same agent (if only one present)

---

## Process of Elimination

### What It Is

Rule out impossible candidates to narrow identity.

### How It Works

**Example:**
```python
# 2 people exit garage
track_1.identity_candidates = {Agent_A, Agent_C}
track_2.identity_candidates = {Agent_A, Agent_C}

# Evidence: track_1 face match = Agent_A
track_1.global_agent_id = Agent_A
track_1.identity_candidates.clear()

# Elimination: Agent_A can't be in track_2
track_2.identity_candidates.remove(Agent_A)
# Only Agent_C remains
track_2.global_agent_id = Agent_C  # Resolved!
```

### When It Applies

- Agent confirmed elsewhere rules them out as source for new track
- Cascading: ruling out candidates increases confidence in remaining
- Eventually resolves to single agent (or stays ambiguous if multiple remain)

**See:** L4_AlibiCheck.md for cascading elimination logic

---

## Portal Handling

### What Is a Portal

Configured location where agents can become concealed (garage door, building entrance)

**Configuration:**
```yaml
portals:
  - name: "garage_door"
    camera_id: "Driveway"
    cells: [(3,2), (3,3)]
    type: "structure_entry"
```

**See:** L4_PortalTransitions.md for portal handling concept, L3_Configuration.md for portal config

---

### Portal Transition Logic

**Condition:** Track ends at portal, new track starts at same portal

**Time doesn't matter:** Agent can be in structure for any duration

**Decision logic:**
```python
if track_A.at_portal == "garage_door" and \
   track_B.at_portal == "garage_door":
    # Plausible transition (time gap irrelevant)
    if high_reID_similarity:
        return "AUTO_MERGE"
    else:
        return "QUEUE_FOR_REVIEW"  # Rely on secondary evidence
```

**Why:** Standard grid travel time logic doesn't apply when crossing portals.

---

## Face Library Safety

### Only High-Confidence Merges Train Face Library

**Condition:** Face recognition ≥ 0.75 threshold OR validated human review

**Why:** Prevents cascading errors (no uncertain merges pollute library)

```python
if merge_decision == "AUTO_MERGE" and \
   (face_confidence >= 0.75 or human_validated):
    face_library.add_training_sample(track, agent_id)
```

**Result:** Face recognition improves as library grows, but only from clean data.

**See:** L13_FaceRecognition.md for library management

---

## Creating New Agents

### When to Create New Agent

**Conditions:**
- Track has no plausible cross-camera candidates
- All merge attempts rejected
- No matching identity in system

**Decision:**
```python
if no_plausible_candidates:
    track.global_agent_id = create_new_agent()
    return "NEW_AGENT"
```

**Result:** New global_agent_id assigned, becomes candidate for future merges.

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - IdentityResolver overview
- **L2_Strategy.md** - Evidence hierarchy, grid learning principles

### Previous Handler
- **L3_TemporalLinking.md** - Links LocalTracks across chunks

### Next Handler
- **L3_EvidenceProcessing.md** - Resolves uncertain tracks when evidence arrives

### L4 Concepts
- **L4_GridLearning.md** - 6×6 grid learning concept (travel times, overlap zones, bootstrap)
- **L4_AlibiCheck.md** - Alibi check and process of elimination
- **L4_PortalTransitions.md** - Portal handling (structure entry, vehicle entry)
- **L4_VehicleOccupancy.md** - Vehicle occupancy tracking

### Algorithms
- **L12_GridLearning.md** - Grid learning algorithm (if needed for detailed pseudocode)
- **L12_AlibiCheck.md** - Alibi check algorithm (if needed for detailed pseudocode)
- **L12_ProcessOfElimination.md** - Cascading elimination logic
- **L12_Merging.md** - Cross-camera merge algorithm

### Implementation
- **L13_FaceRecognition.md** - CodeProject.AI integration, face library management
- **L13_ReID.md** - Re-ID embedding extraction and comparison
- **L14_GridLearningEngine.py** - Grid learning implementation
- **L14_AlibiChecker.py** - Alibi check implementation
- **L14_PortalHandler.py** - Portal transition handling
- **L14_VehicleTracker.py** - Vehicle occupancy tracking
