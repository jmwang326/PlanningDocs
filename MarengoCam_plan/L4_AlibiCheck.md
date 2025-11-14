# Level 4 - Alibi Check (Process of Elimination)

## Purpose
Detailed specification for the alibi check concept - ruling out impossible candidates by verifying they couldn't have reached the entry point at the observed time. This is a key disambiguation technique for cross-camera merging.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md)

**Implemented in:** L14_AlibiChecker.py

---

## Core Concept

### What Is an Alibi Check

**Question:** "Could the entry agent have come from elsewhere?"

**Purpose:** Rule out candidates who were seen elsewhere at times incompatible with reaching the entry point

**Example scenario:**
```
Track_Entry appears at FrontDoor at T=10:35:00
Candidates: Agent_A (last seen Driveway at T=10:34:48)
            Agent_B (last seen Garage at T=10:34:55)

Alibi check:
- Agent_A: Travel time from Driveway to FrontDoor = 12s
  Expected: 4-6s. Too slow (Agent_A may have stopped). POSSIBLE
- Agent_B: Travel time from Garage to FrontDoor = 5s
  Expected: 3-7s. Plausible timing. POSSIBLE

Result: 2 alternative sources (AMBIGUOUS)
```

---

## Alibi Check Algorithm

### Step 1: Identify Candidates

**For the entry track:**
- Find all agents seen on ANY camera in recent time window
- Typical window: Last 30 seconds before entry

```python
candidates = db.query(LocalTrack).filter(
    end_time > (entry_track.start_time - 30),
    end_time < entry_track.start_time,
    camera_id != entry_track.camera_id
).all()
```

---

### Step 2: Compute Travel Times

**For each candidate:**
```python
travel_time = entry_track.start_time - candidate.end_time
expected_time = grid[(candidate.camera_id, entry_track.camera_id)][candidate.exit_cell][entry_track.entry_cell]
variance = grid_variance[(candidate.camera_id, entry_track.camera_id)][candidate.exit_cell][entry_track.entry_cell]
```

**Plausibility check:**
```python
min_time = expected_time - (variance * 1.5)
max_time = expected_time + (variance * 1.5)

if min_time <= travel_time <= max_time:
    # Plausible transition
    return "POSSIBLE"
elif travel_time < min_time:
    # Too fast (impossible)
    return "IMPOSSIBLE"
else:
    # Too slow (may have stopped, still possible)
    return "POSSIBLE"  # Accept delays, reject only impossible speeds
```

**Key decision:** Only reject if travel time is impossibly fast (too slow could mean agent stopped/delayed)

---

### Step 3: Count Alternative Sources

**Count plausible candidates:**
```python
alternative_sources = count(candidates where check == "POSSIBLE")
```

**Decision logic:**
```python
if alternative_sources == 0:
    # No one else could have reached entry point
    # Entry agent MUST be from the current candidate
    return "DEFINITIVE"  # Auto-merge
elif alternative_sources == 1:
    # Only one other possibility
    return "AMBIGUOUS"  # Queue for review (binary choice)
else:
    # Multiple alternative sources
    return "AMBIGUOUS"  # Queue for review (multi-way choice)
```

---

## Alibi Check Integration

### In IdentityResolver

**Workflow:**
1. Track_B appears on Camera_B
2. Find recent tracks from Camera_A (potential source)
3. **Grid timing check:** Is travel time plausible?
   - If NO → reject candidate
   - If YES → proceed to alibi check
4. **Alibi check:** Are there alternative sources?
   - If NO → auto-merge (definitive)
   - If YES → queue for review (ambiguous)

**Example code:**
```python
def resolve_identity(entry_track):
    # Find candidates from previous cameras
    candidates = find_recent_tracks(entry_track)

    for candidate in candidates:
        # Grid timing check
        if not grid_timing_plausible(candidate, entry_track):
            continue  # Skip impossible candidates

        # Alibi check
        alibi_result = alibi_check(candidate, entry_track)

        if alibi_result == "DEFINITIVE":
            # No alternative sources, must be this candidate
            entry_track.global_agent_id = candidate.global_agent_id
            return "AUTO_MERGE"

    # If we reach here, multiple candidates or ambiguous
    entry_track.identity_candidates = set(c.global_agent_id for c in candidates)
    return "QUEUE_FOR_REVIEW"
```

---

## Cascading Process of Elimination

### Concept

**When one agent is confirmed, rule them out from other uncertain tracks at the same time**

**Example:**
```
T=10:35:00 - Two tracks appear:
  Track_1 (FrontDoor): candidates = {Agent_A, Agent_B}
  Track_2 (Backyard):  candidates = {Agent_A, Agent_B}

Evidence arrives: Track_1 face match = Agent_A (confirmed)

Cascading:
  Track_1: global_agent_id = Agent_A (resolved)
  Track_2: remove Agent_A from candidates (can't be in two places)
  Track_2: candidates = {Agent_B} (only one remains)
  Track_2: global_agent_id = Agent_B (auto-resolved!)
```

---

### Cascading Algorithm

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
        # Remove confirmed agent from candidates
        track.identity_candidates.discard(agent_id)

        # If only one candidate remains, resolve
        if len(track.identity_candidates) == 1:
            track.global_agent_id = list(track.identity_candidates)[0]
            track.identity_candidates.clear()

            # Propagate through relationships
            propagate_resolution(track)

            # Recurse: this resolution may trigger more eliminations
            apply_elimination(track)
```

**Result:** Resolving one track can trigger cascade of resolutions

---

## Alibi Check Parameters

### Configuration

**In `marengo.yaml`:**
```yaml
thresholds:
  alibi_check:
    enabled: true
    lookback_window: 30  # seconds (how far back to look for candidates)
    max_alternative_sources: 2  # Queue for review if > 2 alternatives
    timing_tolerance: 1.5  # Variance multiplier (±1.5× stddev)
    reject_too_fast_only: true  # Only reject impossible speeds, accept delays
```

**Timing tolerance:**
- `1.5`: Strict (reject if outside ±1.5× variance)
- `2.0`: Moderate (allow more variance)
- `3.0`: Lenient (accept wide timing range)

**Max alternative sources:**
- `0`: Alibi check must be definitive (no alternatives)
- `1`: Accept binary choice (2 candidates)
- `2`: Accept up to 3-way choice
- `∞`: Always queue (rely on other evidence)

---

## Edge Cases

### Case 1: No Grid Data

**Problem:** Grid not learned for camera pair

**Solution:**
```python
if grid_time == infinity:
    # No grid data, can't do alibi check
    # Fall back to face match or human review
    return "AMBIGUOUS"  # Conservative (queue for review)
```

---

### Case 2: Portal Transitions

**Problem:** Travel time irrelevant (agent can be in structure for any duration)

**Solution:**
```python
if candidate.at_portal and entry_track.at_portal:
    # Portal transition, skip alibi check
    # Rely on face match or Re-ID
    return "AMBIGUOUS"  # Time doesn't matter
```

**See:** L4_PortalTransitions.md for portal handling

---

### Case 3: Multiple Agents, Same Source

**Problem:** Two agents exit Camera_A together, both appear on Camera_B

**Example:**
```
Camera_A: Agent_A and Agent_B exit together at T=10:34:50
Camera_B: Track_1 appears at T=10:34:55
          Track_2 appears at T=10:34:56

Alibi check for Track_1:
- Agent_A: Plausible (5s travel)
- Agent_B: Plausible (5s travel)
- Alternative sources: 1 (Agent_B is alternative to Agent_A)

Result: AMBIGUOUS (binary choice)
```

**Solution:** Queue for review, rely on face match or Re-ID to disambiguate

---

### Case 4: Agent Stopped Mid-Transit

**Problem:** Agent takes longer than expected (stopped to talk, tie shoe, etc.)

**Example:**
```
Expected travel time: 4-6s
Observed: 15s (agent stopped)

Alibi check: Too slow, but still POSSIBLE (delays allowed)
```

**Design choice:** Accept delays (only reject if impossibly fast)

**Why:** Common scenario (people stop), better to accept uncertainty than false reject

---

## Alibi Check Confidence

### Confidence Scoring

**Factors affecting confidence:**
- **Grid completeness:** More learned paths = higher confidence
- **Sample count:** More observations = lower variance = tighter bounds
- **Time window:** Narrower window = fewer alternative sources = higher confidence

**Confidence levels:**
- **High:** Grid well-learned, narrow variance, no alternatives
- **Medium:** Grid partial, moderate variance, 1-2 alternatives
- **Low:** Grid sparse, wide variance, many alternatives

**Usage:**
- High confidence alibi check → auto-merge
- Medium confidence → queue for LLM review
- Low confidence → queue for human review

---

## Alibi Check Metrics

### Performance Indicators

**Definitive rate:**
```python
definitive_rate = count(alibi_check == "DEFINITIVE") / count(all_checks)
```

**Target:** > 60% (after bootstrap)

**Alternative sources distribution:**
- 0 alternatives: X% (definitive, auto-merge)
- 1 alternative: Y% (binary choice, LLM review)
- 2+ alternatives: Z% (complex, human review)

**Ideal distribution:** 60/30/10 (most definitive, some binary, few complex)

---

## Related Documents

### Tactical Context
- **L3_IdentityResolution.md** - Alibi check usage in merging logic
- **L3_EvidenceProcessing.md** - Cascading process of elimination

### Other L4 Concepts
- **L4_GridLearning.md** - Grid provides expected travel times
- **L4_PortalTransitions.md** - Portal transitions bypass alibi check

### Algorithms
- **L12_ProcessOfElimination.md** - Detailed cascading logic
- **L12_AlibiCheck.md** - Pseudocode (if needed)

### Implementation
- **L14_AlibiChecker.py** - Alibi check implementation
- **L14_GridQuery.py** - Grid timing queries
- **L14_IdentityResolver.py** - Integration with merging
