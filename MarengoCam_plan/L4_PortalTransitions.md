# Level 4 - Portal Transitions (Structure & Vehicle Entry)

## Purpose
Detailed specification for handling portal transitions - locations where agents can become concealed (garage doors, building entrances, vehicles). Portals require special merging logic because travel time becomes irrelevant.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_Configuration.md](L3_Configuration.md)

**Implemented in:** L14_PortalHandler.py

---

## Core Concept

### What Is a Portal

**Definition:** A configured location where agents can become concealed from camera view

**Portal types:**
1. **Structure Entry:** Building door, garage entry (person enters structure)
2. **Vehicle Entry:** Car interior (person enters vehicle)

**Key characteristic:** Agent can remain concealed for any duration (seconds to hours)

**Why special handling:** Standard grid timing doesn't apply (time gap irrelevant)

---

## Portal Configuration

### Portal Definition

**In `marengo.yaml`:**
```yaml
portals:
  - id: "garage_door"
    name: "Garage Door"
    camera_id: "Driveway"
    cells: [[3, 2], [3, 3], [4, 2], [4, 3]]  # Grid cells marking portal
    type: "structure_entry"
    bidirectional: true
    metadata:
      structure_name: "Garage"

  - id: "front_door"
    name: "Front Door"
    camera_id: "FrontPorch"
    cells: [[2, 3], [3, 3]]
    type: "structure_entry"
    bidirectional: true
    metadata:
      structure_name: "House"

  - id: "car_driveway"
    name: "Car in Driveway"
    camera_id: "Driveway"
    cells: [[1, 1], [1, 2], [2, 1], [2, 2]]  # Parking spot
    type: "vehicle_entry"
    bidirectional: true
    metadata:
      vehicle_id: "Car_A"  # Link to vehicle track
```

**Fields:**
- `id`: Unique portal identifier
- `camera_id`: Camera that observes this portal
- `cells`: List of [x, y] grid cells marking portal location
- `type`: Portal type (affects merging logic)
- `bidirectional`: Agent can enter AND exit (default: true)
- `metadata`: Portal-specific data (structure name, vehicle ID, etc.)

**See:** L3_Configuration.md for full configuration syntax

---

## Portal Types

### Type 1: Structure Entry

**Scenarios:**
- Person walks into garage (exits Driveway camera)
- Person exits garage hours later (re-enters Driveway camera)
- Person enters front door (exits FrontPorch camera)
- Person exits front door (re-enters FrontPorch camera)

**Characteristics:**
- **Time gap:** Irrelevant (agent can be inside for any duration)
- **Same camera:** Entry and exit typically on same camera
- **Bidirectional:** Agent can enter and exit through same portal

**Merging logic:**
```python
if track_A.at_portal == "garage_door" and \
   track_B.at_portal == "garage_door" and \
   track_A.camera_id == track_B.camera_id:
    # Same portal, same camera
    # Time gap doesn't matter

    if high_reid_similarity(track_A, track_B):
        # Appearance matches (clothing, body shape)
        return "AUTO_MERGE"
    elif face_match(track_A, track_B) >= 0.75:
        # Face match definitive
        return "AUTO_MERGE"
    else:
        # Rely on secondary evidence
        return "QUEUE_FOR_REVIEW"
```

---

### Type 2: Vehicle Entry

**Scenarios:**
- Person walks to car, enters vehicle (exits Driveway camera)
- Vehicle drives to different location
- Person exits vehicle (appears on different camera)

**Characteristics:**
- **Time gap:** Irrelevant (vehicle can travel any distance)
- **Cross-camera:** Exit and entry typically on different cameras
- **Vehicle tracking:** Requires vehicle occupancy tracking

**Merging logic:**
```python
if track_A.at_portal == "car_driveway" and \
   track_B.at_portal and track_B.portal_metadata.vehicle_id == "Car_A":
    # Person entered vehicle on Camera_A, vehicle moved, person exits on Camera_B

    # Requires vehicle occupancy tracking
    occupancy = get_vehicle_occupancy("Car_A", track_A.end_time, track_B.start_time)

    if occupancy == 1:
        # Only one person in vehicle
        return "AUTO_MERGE"  # Must be same person
    elif occupancy > 1:
        # Multiple occupants (driver + passenger)
        return "QUEUE_FOR_REVIEW"  # Ambiguous
    else:
        # Occupancy unknown
        return "QUEUE_FOR_REVIEW"  # Conservative
```

**See:** L4_VehicleOccupancy.md for vehicle tracking details

---

## Portal Detection

### Marking Tracks at Portals

**During ChunkProcessor:**
```python
def mark_portal(track):
    # Check if track's exit cell overlaps with any portal
    for portal in config.portals:
        if track.camera_id == portal.camera_id and \
           track.exit_cell in portal.cells:
            # Track exited at portal
            track.at_portal = portal.id
            track.portal_type = portal.type
            track.portal_metadata = portal.metadata
            return

    # No portal detected
    track.at_portal = None
```

**Track data model:**
```python
LocalTrack(
    # ... other fields
    at_portal: Optional[str],  # Portal ID if track ended at portal
    portal_type: Optional[str],  # "structure_entry" or "vehicle_entry"
    portal_metadata: Optional[dict],  # Portal-specific data
)
```

---

## Portal Transition Merging

### Merging Logic

**Step 1: Detect portal transition**
```python
def is_portal_transition(track_A, track_B):
    if track_A.at_portal and track_B.at_portal:
        # Both tracks at portals
        if track_A.at_portal == track_B.at_portal:
            # Same portal (e.g., garage exit → garage entry)
            return True
        elif track_A.portal_type == "vehicle_entry" and \
             track_B.portal_type == "vehicle_entry" and \
             track_A.portal_metadata.vehicle_id == track_B.portal_metadata.vehicle_id:
            # Same vehicle (different portals)
            return True

    return False
```

**Step 2: Apply portal-specific merging**
```python
def merge_portal_transition(track_A, track_B):
    if track_A.portal_type == "structure_entry":
        # Structure entry/exit (same camera)
        return merge_structure_portal(track_A, track_B)
    elif track_A.portal_type == "vehicle_entry":
        # Vehicle entry/exit (cross-camera)
        return merge_vehicle_portal(track_A, track_B)
```

---

### Structure Portal Merging

**Evidence hierarchy (portal transitions):**
1. **Face match** (≥ 0.75) → auto-merge
2. **Re-ID similarity** (≥ 0.80) + same clothing → auto-merge
3. **Unique entry** (only one agent entered portal) → auto-merge
4. **Ambiguous** → queue for review

```python
def merge_structure_portal(track_A, track_B):
    # Face match
    if face_match(track_A, track_B) >= 0.75:
        return "AUTO_MERGE"

    # Re-ID similarity (high threshold for portal)
    if reid_similarity(track_A, track_B) >= 0.80:
        return "AUTO_MERGE"

    # Unique entry check
    recent_entries = count_portal_entries(track_A.at_portal, track_A.end_time - 60, track_B.start_time)
    if recent_entries == 1:
        # Only one person entered, must be same person exiting
        return "AUTO_MERGE"

    # Ambiguous
    return "QUEUE_FOR_REVIEW"
```

---

### Vehicle Portal Merging

**Requires vehicle occupancy tracking:**

```python
def merge_vehicle_portal(track_A, track_B):
    vehicle_id = track_A.portal_metadata.vehicle_id

    # Check vehicle occupancy
    occupancy = get_vehicle_occupancy(vehicle_id, track_A.end_time, track_B.start_time)

    if occupancy == 1:
        # Only one person in vehicle
        return "AUTO_MERGE"  # Definitive
    elif occupancy > 1:
        # Multiple occupants
        # Rely on face match or Re-ID
        if face_match(track_A, track_B) >= 0.75:
            return "AUTO_MERGE"
        elif reid_similarity(track_A, track_B) >= 0.80:
            return "AUTO_MERGE"
        else:
            return "QUEUE_FOR_REVIEW"
    else:
        # Occupancy unknown
        return "QUEUE_FOR_REVIEW"  # Conservative
```

**See:** L4_VehicleOccupancy.md for occupancy tracking details

---

## Portal-Specific Challenges

### Challenge 1: Long Time Gaps

**Problem:** Agent enters garage, emerges hours later

**Example:**
```
Track_A: Exit garage at T=08:00:00
Track_B: Enter garage at T=18:30:00 (10.5 hours later)
```

**Solution:**
- Don't reject based on time gap
- Rely heavily on Re-ID (clothing may have changed)
- Face match still reliable
- Queue for review if ambiguous

---

### Challenge 2: Multiple People, Same Portal

**Problem:** Three people enter garage together, exit at different times

**Example:**
```
T=08:00:00: Agent_A, Agent_B, Agent_C enter garage (3 tracks)
T=08:05:00: One person exits (Track_4)
T=08:10:00: One person exits (Track_5)
T=08:15:00: One person exits (Track_6)

Question: Which entry → which exit?
```

**Solution:**
- Multi-assignment: Track_4 candidates = {Agent_A, Agent_B, Agent_C}
- Rely on face match or Re-ID to disambiguate
- Process of elimination (if Agent_A confirmed in Track_4, remove from Track_5/Track_6)
- Queue complex cases for review

---

### Challenge 3: Clothing Change

**Problem:** Agent enters garage wearing one outfit, exits wearing different outfit

**Example:**
```
Track_A: Enter garage (blue shirt)
Track_B: Exit garage (red shirt)

Re-ID similarity: 0.25 (low, different clothing)
Face match: 0.88 (high, same person)
```

**Solution:**
- Face match overrides Re-ID (clothing can change in structure)
- Don't rely on Re-ID alone for structure portals
- Accept lower Re-ID similarity for long time gaps (> 30 min)

---

## Portal Learning

### Unique Entry Patterns

**Learn:**
- How often portals are used (high-traffic vs rare)
- Typical dwell time (how long agents stay in structure)
- Typical occupancy (solo vs groups)

**Usage:**
- Prioritize high-traffic portals for configuration
- Adjust thresholds based on dwell time distribution
- Flag unusual patterns (e.g., person enters but never exits)

---

### Portal Validation

**Sanity checks:**
- Every entry should have corresponding exit (eventually)
- Exit count ≈ entry count (over time)
- Flag if unbalanced (possible configuration error)

**Example alert:**
```
Portal "garage_door": 47 entries, 32 exits in last 24 hours
Possible issue: misconfigured portal cells or missing camera coverage
```

---

## Portal UI

### Configuration UI

**Features:**
- **Grid overlay:** Drag-and-drop to mark portal cells
- **Portal types:** Select structure_entry or vehicle_entry
- **Bidirectional:** Toggle enter/exit
- **Metadata:** Link to vehicle (for vehicle portals)

**Visualization:**
- Highlight portal cells on camera grid
- Show entry/exit statistics
- Display unbalanced portals (warning)

**See:** L4_ConfigurationUI.md for UI specification

---

### Monitoring UI

**Dashboard metrics:**
- Portal usage frequency (entries/exits per hour)
- Average dwell time (time between entry and exit)
- Unbalanced portals (entry count != exit count)
- Uncertain transitions (queue count for portal merges)

**See:** L4_HealthDashboard.md for dashboard specification

---

## Related Documents

### Tactical Context
- **L3_IdentityResolution.md** - Portal handling in merging logic
- **L3_Configuration.md** - Portal configuration syntax
- **L3_ChunkProcessing.md** - Portal detection during tracking

### Other L4 Concepts
- **L4_GridLearning.md** - Grid timing bypassed for portals
- **L4_AlibiCheck.md** - Alibi check not applicable to portals
- **L4_VehicleOccupancy.md** - Vehicle tracking for vehicle portals
- **L4_ConfigurationUI.md** - Portal configuration interface

### Implementation
- **L14_PortalHandler.py** - Portal detection, merging logic
- **L14_PortalMonitor.py** - Entry/exit validation, metrics
