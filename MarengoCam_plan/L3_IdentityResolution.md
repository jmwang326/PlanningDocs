# Level 3 - Identity Resolution (IdentityResolver)

## Purpose
Assign global_agent_id to LocalTracks (cross-camera merging).

---

## Input/Output

**Input:** LocalTrack without global_agent_id (after TemporalLinker)

**Output:** global_agent_id (certain) OR identity_candidates (uncertain)

**Timing:** Periodic batch processing (every ~30s)

---

## Evidence Hierarchy

### 1. Face Recognition (Definitive)

**Method:** CodeProject.AI face library

**Threshold:** ≥ 0.75

**Decision:** AUTO-MERGE

```python
if face_match_confidence >= 0.75:
    track.global_agent_id = matched_agent_id
```

---

### 2. Grid Overlap (Definitive if Single Agent)

**Condition:** Travel time < 0.4s

**Meaning:** Cameras see overlapping space (same physical area)

```python
if travel_time < 0.4 and only_one_agent_visible:
    track_B.global_agent_id = track_A.global_agent_id
```

**Key insight:** No time gap = same space = same agent (if only one present).

---

### 3. Grid Transition + Alibi Check

**Condition:** Travel time fits learned 6×6 grid pattern

```python
travel_time = track_B.start_time - track_A.end_time
grid_time = grid.get_time(camera_A, camera_B, exit_cell_A, entry_cell_B)

if abs(travel_time - grid_time) < variance_threshold:
    if alibi_check_passes(track_B, travel_time):
        # No other agents could have reached track_B
        track_B.global_agent_id = track_A.global_agent_id
    else:
        # Alternative sources exist
        track_B.identity_candidates = {Agent_A, Agent_X, ...}
```

**Alibi check:** "Could entry agent have come from elsewhere?"
- Check all cameras: could any other recent track have arrived at this entry point?
- If NO → must be from track_A (process of elimination)
- If YES → ambiguous (multiple candidates)

---

### 4. LLM / Human Adjudication (Fallback)

**Condition:** Time/space plausible but ambiguous

**Decision:** QUEUE_FOR_REVIEW

```python
track.identity_candidates = {Agent_A, Agent_B, ...}
queue_for_review(track, reason="ambiguous_timing")
```

---

### 5. Reject (Definitive)

**Conditions:**
- Face mismatch (different person, high confidence)
- Spatial/temporal impossibility (travel time too short/long)
- Alibi check fails (seen elsewhere simultaneously)

---

## Grid-Based Spatial Learning (6×6)

**Per camera:** 6×6 grid cells overlaid on FOV

**Per camera pair:** Learned minimum travel times between exit/entry cells

**Example:**
```python
grid[CameraA→CameraB][(5,2)→(0,3)] = 1.5s
```

**Meaning:** Fastest observed time from CameraA cell (5,2) to CameraB cell (0,3) is 1.5s.

**How learning works:**
1. Person walks from Camera A to Camera B (validated merge)
2. Measure: `travel_time = track_B.start_time - track_A.end_time`
3. Map: `exit_cell = track_A.exit_cell`, `entry_cell = track_B.entry_cell`
4. Learn: `if travel_time < current: grid[A→B][exit→entry] = travel_time`

**What gets learned (clean data only):**
- Solo tracks (max_concurrent_agents = 1)
- High-confidence merges (face ≥ 0.75 OR validated human review)
- Sustained movement (≥ 3s duration)

**Exclude:**
- Groups (multiple people) - ambiguous which person traveled
- Ambiguous merges
- Low-quality tracks

**Overlap zones:** Travel time < 0.4s marks cameras seeing overlapping space.

---

## Process of Elimination

Rule out impossible candidates.

**Example:**
```python
# 2 people exit garage
track_1.identity_candidates = {Agent_A, Agent_C}
track_2.identity_candidates = {Agent_A, Agent_C}

# Evidence: track_1 face match = Agent_A
track_1.global_agent_id = Agent_A

# Elimination: Agent_A can't be in track_2
track_2.identity_candidates.remove(Agent_A)
# Only Agent_C remains
track_2.global_agent_id = Agent_C  # Resolved!
```

**Cascading:** Resolving one track can cascade through entire timeline.

---

## Portal Handling

**Portal:** Configured location where agents become concealed (garage door, building entrance).

```yaml
portals:
  - name: "garage_door"
    camera_id: "Driveway"
    cells: [(3,2), (3,3)]
    type: "structure_entry"
```

**Portal transition logic:**

Track ends at portal, new track starts at same portal → time doesn't matter (agent can be inside for any duration).

```python
if track_A.at_portal == "garage_door" and \
   track_B.at_portal == "garage_door":
    # Plausible transition (time gap irrelevant)
    if high_reID_similarity:
        return "AUTO_MERGE"
    else:
        return "QUEUE_FOR_REVIEW"
```

---

## Face Library Safety

**Only high-confidence merges train face library:**
- Face recognition ≥ 0.75 OR validated human review

```python
if merge_decision == "AUTO_MERGE" and \
   (face_confidence >= 0.75 or human_validated):
    face_library.add_training_sample(track, agent_id)
```

**Result:** Library improves from clean data only (no pollution).

---

## Creating New Agents

**When:**
- Track has no plausible cross-camera candidates
- All merge attempts rejected
- No matching identity in system

```python
if no_plausible_candidates:
    track.global_agent_id = create_new_agent()
```
