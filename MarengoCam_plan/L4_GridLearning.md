# Level 4 - Grid Learning (6×6 Spatial Model)

## Purpose
Detailed specification for the 6×6 grid-based spatial learning system that learns minimum travel times between cameras. This concept enables automated cross-camera merging based on spatial plausibility.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_Configuration.md](L3_Configuration.md)

**Implemented in:** L14_GridLearningEngine.py

---

## Core Concept

### What Is the 6×6 Grid

**Per camera:** Overlay a 6×6 grid on the camera's field of view (FOV)

**Grid cells:** Each cell is identified by (x, y) coordinates (0-5, 0-5)

**Example visualization:**
```
Camera FOV with 6×6 grid:
┌─────┬─────┬─────┬─────┬─────┬─────┐
│ 0,5 │ 1,5 │ 2,5 │ 3,5 │ 4,5 │ 5,5 │
├─────┼─────┼─────┼─────┼─────┼─────┤
│ 0,4 │ 1,4 │ 2,4 │ 3,4 │ 4,4 │ 5,4 │
├─────┼─────┼─────┼─────┼─────┼─────┤
│ 0,3 │ 1,3 │ 2,3 │ 3,3 │ 4,3 │ 5,3 │
├─────┼─────┼─────┼─────┼─────┼─────┤
│ 0,2 │ 1,2 │ 2,2 │ 3,2 │ 4,2 │ 5,2 │
├─────┼─────┼─────┼─────┼─────┼─────┤
│ 0,1 │ 1,1 │ 2,1 │ 3,1 │ 4,1 │ 5,1 │
├─────┼─────┼─────┼─────┼─────┼─────┤
│ 0,0 │ 1,0 │ 2,0 │ 3,0 │ 4,0 │ 5,0 │
└─────┴─────┴─────┴─────┴─────┴─────┘
```

**Grid calibration:**
- **Auto:** System divides FOV into equal 6×6 cells
- **Manual:** User adjusts grid to match physical layout (in configuration UI)

---

### What Gets Learned

**Per camera pair:** Learn minimum travel time between exit/entry cells

**Data structure:**
```python
grid[(CameraA, CameraB)][(exit_cell_x, exit_cell_y)][(entry_cell_x, entry_cell_y)] = min_travel_time
```

**Example:**
```python
# Fastest observed time from Driveway cell (5,2) to FrontDoor cell (0,3)
grid[("Driveway", "FrontDoor")][(5, 2)][(0, 3)] = 4.2  # seconds
```

**Interpretation:** An agent exiting Driveway at cell (5,2) can reach FrontDoor cell (0,3) in 4.2 seconds minimum.

---

## Learning Process

### 1. Observation

**Trigger:** Validated merge between tracks on different cameras

**Requirements:**
- Track A (Camera A, exit cell known)
- Track B (Camera B, entry cell known)
- Merge confirmed (face match ≥ 0.75 OR human validation)

### 2. Measurement

**Travel time calculation:**
```python
travel_time = track_B.start_time - track_A.end_time
```

**Entry/exit cell extraction:**
```python
exit_cell = track_A.exit_cell  # Last grid cell before leaving Camera A
entry_cell = track_B.entry_cell  # First grid cell when appearing on Camera B
```

### 3. Learning Update

**Update rule:** Keep minimum travel time

```python
current_time = grid[(camera_A, camera_B)][exit_cell][entry_cell]

if travel_time < current_time:
    # New minimum observed
    grid[(camera_A, camera_B)][exit_cell][entry_cell] = travel_time
```

**Why minimum:** Fastest observed time represents most direct path (no delays)

**Variance tracking:** Also track observed variance for plausibility checks
```python
variance[(camera_A, camera_B)][exit_cell][entry_cell] = stddev(observed_times)
```

---

### 4. Clean Data Only

**Include in learning:**
- ✅ Solo tracks (max_concurrent_agents = 1)
- ✅ High-confidence merges (face match ≥ 0.75 OR human validated)
- ✅ Sustained movement (≥ 3s duration)
- ✅ Clean observations (no occlusions, good lighting)

**Exclude from learning:**
- ❌ Groups (multiple people together) - ambiguous which person traveled
- ❌ Ambiguous merges (uncertain identity)
- ❌ Sporadic vehicle detections (person in car)
- ❌ Low-quality tracks (partial visibility)
- ❌ Portal transitions (time gap irrelevant)

**Why:** Prevents noise pollution. Grid learns only from reliable, unambiguous data.

---

## Using Learned Grid

### Plausibility Check

**When merging tracks, check if travel time is plausible:**

```python
observed_travel_time = track_B.start_time - track_A.end_time
expected_time = grid[(camera_A, camera_B)][exit_A][entry_B]
variance = grid_variance[(camera_A, camera_B)][exit_A][entry_B]

if abs(observed_travel_time - expected_time) < variance * 1.5:
    # Plausible transition (within tolerance)
    return "PLAUSIBLE"
else:
    # Too fast or too slow
    return "IMPLAUSIBLE"
```

**Tolerance:** Typically ±30% or 1.5× variance (whichever is larger)

**Example:**
- Expected: 4.0s
- Variance: 0.8s
- Accept: 2.8s - 5.2s (4.0 ± 1.2s)

---

### Overlap Zones

**Special case:** Travel time < 0.4s

**What it means:** Cameras see overlapping physical space (same location)

**Grid marking:**
```python
grid[(camera_A, camera_B)][exit_cell][entry_cell] = 0  # Overlap zone
```

**Merging logic:**
```python
if grid_time == 0 and only_one_agent_visible:
    # Must be same agent (no time gap = same space)
    return "AUTO_MERGE"
elif grid_time == 0 and multiple_agents_visible:
    # Overlap zone but ambiguous (multiple candidates)
    return "QUEUE_FOR_REVIEW"
```

**Key insight:** No time gap = same physical space = same agent (if only one present)

---

## Grid Completeness

### Bootstrap Phase

**Initial state:** All grid cells initialized to `infinity` (no learned paths)

**Progress metric:**
```python
grid_completeness = count(grid[*][*][*] < infinity) / total_possible_paths
```

**Example:**
- Total paths: 10 cameras × 10 cameras × 36 exit cells × 36 entry cells = 129,600 paths
- Learned paths: 10,368
- Completeness: 8%

**Target:** > 80% completeness for common paths (not all 129k paths needed)

---

### Common Paths

**Focus learning on:**
- High-traffic paths (frequently traveled routes)
- Adjacent camera pairs (neighbors)
- Portal transitions (garage to driveway, etc.)

**Rare paths:**
- Distant cameras rarely directly connected
- Many theoretical paths never observed
- Accept uncertainty for rare transitions

---

## Grid Visualization

**See:** L4_GridVisualization.md for UI specification

**Key features:**
- Heatmap showing learned travel times
- Highlight overlap zones (< 0.4s)
- Show completeness per camera pair
- Identify missing/rare paths

---

## Grid Storage

### Data Structure

**In-memory:**
```python
grid: Dict[
    Tuple[CameraID, CameraID],  # Camera pair
    Dict[
        Tuple[int, int],  # Exit cell (x, y)
        Dict[
            Tuple[int, int],  # Entry cell (x, y)
            float  # Min travel time (seconds)
        ]
    ]
]
```

**Persistence:** Store in database for durability

```sql
CREATE TABLE grid_transitions (
    camera_from VARCHAR(50),
    camera_to VARCHAR(50),
    exit_cell_x INT,
    exit_cell_y INT,
    entry_cell_x INT,
    entry_cell_y INT,
    min_travel_time FLOAT,
    variance FLOAT,
    sample_count INT,
    last_updated TIMESTAMP,
    PRIMARY KEY (camera_from, camera_to, exit_cell_x, exit_cell_y, entry_cell_x, entry_cell_y)
);
```

---

## Bootstrap Strategy

### Phase 1: No Grid Data (Days 1-3)

**Behavior:**
- All cross-camera merges require face match (≥ 0.75)
- OR human/LLM review
- No grid-based auto-merge

**Goal:** Gather initial observations

---

### Phase 2: Partial Grid (Days 4-14)

**Behavior:**
- Use learned paths where available
- Fall back to face match or review for unlearned paths
- Auto-merge on overlap zones (< 0.4s)

**Goal:** Build grid coverage

---

### Phase 3: Complete Grid (Days 15+)

**Behavior:**
- Grid-based auto-merge for most transitions
- Face match for disambiguation
- Rare paths still require review

**Goal:** High autonomy (80%+ auto-merge rate)

---

## Edge Cases

### New Camera Added

**Problem:** No grid data for new camera

**Solution:**
- Initialize all paths to infinity
- Collect observations (bootstrap mode)
- Gradual learning over 3-7 days

### Camera Moved/Recalibrated

**Problem:** Existing grid data invalid

**Solution:**
- Clear grid for affected camera pairs
- Re-learn from scratch
- Flag in configuration (triggers reset)

### Seasonal Changes

**Problem:** Travel times change (snow, construction, etc.)

**Solution:**
- Grid updates continuously (always keeps minimum)
- Variance adapts to new observations
- No explicit seasonality handling (just adapts)

---

## Related Documents

### Tactical Context
- **L3_IdentityResolution.md** - How grid learning is used for merging
- **L3_Configuration.md** - Grid configuration (6×6 size, calibration)

### Other L4 Concepts
- **L4_AlibiCheck.md** - Uses grid to check alternative sources
- **L4_PortalTransitions.md** - Portal transitions ignore grid timing
- **L4_GridVisualization.md** (planned) - Grid visualization tool

### Algorithms
- **L12_GridLearning.md** - Detailed learning algorithm (if needed for pseudocode)

### Implementation
- **L14_GridLearningEngine.py** - Learning updates, storage
- **L14_GridQuery.py** - Plausibility checks, overlap detection
- **L13_Database.md** - Grid storage schema
