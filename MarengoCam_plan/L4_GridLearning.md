# Level 4 - Grid Learning (6×6 Spatial Model)

## Purpose
Specification for the 6×6 grid-based spatial learning system that learns minimum travel times between cameras.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_Configuration.md](L3_Configuration.md)

**Implemented in:** L14_GridLearningEngine.py

---

## Concept

### The 6×6 Grid

**Per camera:** Overlay a 6×6 grid on the camera's field of view

**What gets learned:** Minimum travel time between (camera_A, exit_cell) → (camera_B, entry_cell)

**Example:**
```python
grid[("Driveway", "FrontDoor")][(5, 2)][(0, 3)] = 4.2  # seconds
```

**Meaning:** Fastest observed time from Driveway cell (5,2) to FrontDoor cell (0,3) is 4.2 seconds.

---

## Learning Process

**1. Observation:** Validated merge between tracks on different cameras (face match ≥ 0.75 OR human validation)

**2. Measurement:**
```python
travel_time = track_B.start_time - track_A.end_time
exit_cell = track_A.exit_cell
entry_cell = track_B.entry_cell
```

**3. Update:** Keep minimum
```python
if travel_time < grid[(camera_A, camera_B)][exit_cell][entry_cell]:
    grid[(camera_A, camera_B)][exit_cell][entry_cell] = travel_time
```

---

## Clean Data Only

**Include:**
- Solo tracks (max_concurrent_agents = 1)
- High-confidence merges (face match ≥ 0.75 OR human validated)
- Sustained movement (≥ 3s duration)

**Exclude:**
- Groups (ambiguous which person traveled)
- Ambiguous merges
- Portal transitions (time gap irrelevant)

---

## Usage

**Plausibility check:**
```python
observed = track_B.start_time - track_A.end_time
expected = grid[(camera_A, camera_B)][exit_A][entry_B]
variance = grid_variance[(camera_A, camera_B)][exit_A][entry_B]

if abs(observed - expected) < variance * 1.5:
    return "PLAUSIBLE"
```

**Overlap zones:** Travel time < 0.4s means cameras see same physical space
```python
if grid_time < 0.4 and only_one_agent_visible:
    return "AUTO_MERGE"  # Must be same agent
```

---

## Bootstrap

**Phase 1 (Days 1-3):** No grid data, require face match or review

**Phase 2 (Days 4-14):** Partial grid, use where available

**Phase 3 (Days 15+):** Grid complete, 80%+ auto-merge rate

**Completeness metric:**
```python
completeness = count(learned_paths) / count(common_paths)
```

Target: > 80% for common paths (not all 129k theoretical paths)

---

## Related Documents

**Tactical:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_Configuration.md](L3_Configuration.md)

**Algorithms:** L12_GridLearning.md (detailed pseudocode if needed)

**Implementation:** L14_GridLearningEngine.py, L14_GridQuery.py
