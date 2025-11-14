# Level 4 - Portal Transitions (Structure & Vehicle Entry)

## Purpose
Specification for handling portal transitions - locations where agents can become concealed. Portals require special merging logic because travel time becomes irrelevant.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_Configuration.md](L3_Configuration.md)

**Implemented in:** L14_PortalHandler.py

---

## Concept

**Definition:** Configured location where agents can become concealed from camera view

**Key characteristic:** Agent can remain concealed for any duration (seconds to hours)

**Why special:** Standard grid timing doesn't apply

---

## Portal Types

### Structure Entry

**Examples:** Building door, garage entry

**Characteristics:**
- Time gap irrelevant (agent inside structure)
- Same camera (entry and exit on same camera)
- Bidirectional (can enter and exit)

**Merging:**
```python
if track_A.at_portal == "garage_door" and \
   track_B.at_portal == "garage_door":
    # Time gap doesn't matter
    if face_match(A, B) >= 0.75:
        return "AUTO_MERGE"
    elif reid_similarity(A, B) >= 0.80:
        return "AUTO_MERGE"
    elif count_portal_entries(60s) == 1:
        return "AUTO_MERGE"  # Only one person entered
    else:
        return "QUEUE_FOR_REVIEW"
```

---

### Vehicle Entry

**Examples:** Person enters car, car drives to different camera, person exits

**Characteristics:**
- Time gap irrelevant (vehicle can travel any distance)
- Cross-camera (exit and entry on different cameras)
- Requires vehicle occupancy tracking

**Merging:**
```python
if track_A.at_portal (vehicle) and track_B.at_portal (vehicle):
    occupancy = get_vehicle_occupancy(vehicle_id, time_window)

    if occupancy == 1:
        return "AUTO_MERGE"  # Only one person in vehicle
    else:
        # Multiple occupants, use face/ReID
        if face_match(A, B) >= 0.75:
            return "AUTO_MERGE"
        else:
            return "QUEUE_FOR_REVIEW"
```

**See:** L4_VehicleOccupancy.md for vehicle tracking

---

## Portal Configuration

```yaml
portals:
  - id: "garage_door"
    name: "Garage Door"
    camera_id: "Driveway"
    cells: [[3, 2], [3, 3], [4, 2], [4, 3]]
    type: "structure_entry"
    bidirectional: true

  - id: "car_driveway"
    name: "Car in Driveway"
    camera_id: "Driveway"
    cells: [[1, 1], [1, 2], [2, 1], [2, 2]]
    type: "vehicle_entry"
    metadata:
      vehicle_id: "Car_A"
```

**See:** L3_Configuration.md for full config syntax

---

## Portal Detection

**Mark tracks at portals:**
```python
def mark_portal(track):
    for portal in config.portals:
        if track.camera_id == portal.camera_id and \
           track.exit_cell in portal.cells:
            track.at_portal = portal.id
            track.portal_type = portal.type
            return
```

---

## Evidence Hierarchy (Portals)

**For structure portals:**
1. Face match (≥ 0.75) → auto-merge
2. Re-ID similarity (≥ 0.80) → auto-merge
3. Unique entry (only one person entered) → auto-merge
4. Ambiguous → queue for review

**For vehicle portals:**
1. Single occupant + time match → auto-merge
2. Multiple occupants + face match (≥ 0.75) → auto-merge
3. Ambiguous → queue for review

---

## Edge Cases

**Long time gaps:** Accept (person can be inside for hours)

**Multiple people, same portal:** Multi-assignment, use face/ReID to disambiguate

**Clothing change:** Face match overrides Re-ID (clothing can change in structure)

---

## Portal Metrics

**Entry/exit balance:** Track entries ≈ exits (over time)

**Typical dwell time:** Learn how long agents stay in structure

**Alert if unbalanced:** Many entries, few exits → possible config error

---

## Related Documents

**Tactical:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_Configuration.md](L3_Configuration.md)

**Other L4:** L4_GridLearning.md (bypassed for portals), L4_AlibiCheck.md (not applicable), L4_VehicleOccupancy.md

**Implementation:** L14_PortalHandler.py
