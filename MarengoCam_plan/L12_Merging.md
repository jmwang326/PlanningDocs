# Level 12 - Merging Technical Specs

## Purpose
Technical implementation of cross-camera merge logic.

## Merge Algorithm

### Candidate Finding
**Runs periodically (every 30s):**
```python
def find_merge_candidates():
    recent_tracks = get_tracks(last_60_seconds)
    for track1 in recent_tracks:
        for track2 in recent_tracks:
            if time_space_plausible(track1, track2):
                if portal_connected(track1.camera, track2.camera):
                    yield (track1, track2)
```

### Time/Space Gating
**Filters impossible transitions:**
- Temporal: `track2.start_time - track1.end_time < max_window`
- Spatial: distance / time_delta within walking speed (~1 m/s)
- Portal check: track endpoints near portal locations

### Evidence Scoring
**Count evidence, threshold:**
```python
strong_evidence = 0
weak_evidence = 0

if face_match(track1, track2) > 0.75:
    strong_evidence += 1
if only_candidate(track2):
    strong_evidence += 1
if portal_crossing(track1, track2):
    strong_evidence += 1
if spatial_proximity(track1, track2):
    weak_evidence += 1

if strong_evidence >= 2:
    return AUTO_MERGE
elif strong_evidence >= 1 and weak_evidence >= 1:
    return REVIEW_LLM
else:
    return REVIEW_HUMAN
```

## Face Recognition

### CodeProject.AI Integration
**Endpoint:**
```
POST http://{codeproject_server}/v1/vision/face/recognize
```

**Request:**
```json
{
  "image": "base64_encoded_jpeg",
  "min_confidence": 0.60
}
```

**Response:**
```json
{
  "predictions": [
    {
      "userid": "person_a",
      "confidence": 0.85
    }
  ]
}
```

### Face Library Management
- Register face: POST to `/v1/vision/face/register`
- Delete face: DELETE `/v1/vision/face/{userid}`
- List faces: GET `/v1/vision/face/list`

## LLM Adjudication

### Gemini Flash 2.0
**Endpoint:** Google AI Studio API

**Prompt template:**
```
Are these the same person/vehicle?

Image Set A:
{panel_a_images}

Image Set B:
{panel_b_images}

Context:
- Camera A → Camera B
- Time gap: {time_delta} seconds
- Portal: {portal_name}

Respond in JSON:
{
  "answer": "YES|NO|CANNOT",
  "confidence": 0.0-1.0,
  "reasons": "brief explanation"
}
```

### Panel Assembly
**2×6 layout (768×768 pixels):**
- Top row: Track A best frames (K=3)
- Bottom row: Track B best frames (K=3)
- Quality scoring: blur, brightness, face-visible

## Multi-Assignment

### Data Structure
**Track can link to multiple agents:**
```python
class Track:
    agent_assignments: List[AgentAssignment]

class AgentAssignment:
    agent_id: int
    evidence: List[str]  # ["face_match", "portal_crossing"]
    validated: bool
    # Note: No probabilities (see L2D_Strategy - no percentages)
```

### Uncertainty Persistence
- Assignments persist until validated (human/LLM)
- Timeline queries return all possibilities
- UI shows uncertainty ("possibly Agent A or B")

## Vehicle Association Technical Specs

### Agent State Structure
**Person agent:**
```python
class PersonAgent:
    agent_id: int
    state: Literal["visible", "in_vehicle", "in_structure", "unknown"]
    vehicle_id: Optional[int]  # If state == "in_vehicle"
    last_track_id: int
    last_seen_timestamp: float
```

**Vehicle agent:**
```python
class VehicleAgent:
    agent_id: int
    state: Literal["parked", "moving"]
    occupants: List[OccupantAssignment]

class OccupantAssignment:
    agent_id: int
    entry_timestamp: float
    entry_track_id: int
    sporadic_detections: List[Detection]  # Face detections through windows
    validated: bool
```

### Entry Detection Algorithm
```python
def detect_vehicle_entry(person_track, vehicle_track):
    """
    Correlate person disappearance with vehicle movement
    """
    # Temporal correlation
    time_delta = vehicle_track.start_time - person_track.end_time
    if time_delta < 0 or time_delta > 5.0:  # seconds
        return False

    # Spatial correlation
    distance = euclidean_distance(
        person_track.end_position,
        vehicle_track.start_position
    )
    if distance > VEHICLE_PROXIMITY_THRESHOLD:  # e.g., 5 meters
        return False

    # Update states
    person_agent.state = "in_vehicle"
    person_agent.vehicle_id = vehicle_track.agent_id
    vehicle_agent.occupants.append(OccupantAssignment(
        agent_id=person_agent.id,
        entry_timestamp=person_track.end_time,
        entry_track_id=person_track.id
    ))

    return True
```

### Sporadic Interior Detection Handling
```python
def process_interior_detection(vehicle_track, detection):
    """
    Handle person detection inside vehicle (through windows)

    Don't require frame-to-frame continuity
    """
    if detection.class_name != "person":
        return

    # Attempt face match against vehicle occupants
    for occupant in vehicle_agent.occupants:
        face_similarity = face_recognition.compare(
            detection.crop,
            occupant.agent.face_library
        )
        if face_similarity > 0.75:
            # Strengthen assignment
            occupant.sporadic_detections.append(detection)
            occupant.validated = True  # Face match confirms
            return

    # Unknown face in vehicle - possible additional occupant
    # Log but don't force assignment
```

### Exit Detection Algorithm
```python
def detect_vehicle_exit(vehicle_track, person_track):
    """
    Correlate vehicle parking with person appearance
    """
    # Check vehicle is parked
    if vehicle_agent.state != "parked":
        return None

    # Temporal correlation
    time_delta = person_track.start_time - vehicle_track.end_time
    if time_delta < 0 or time_delta > 10.0:
        return None

    # Spatial correlation
    distance = euclidean_distance(
        vehicle_track.end_position,
        person_track.start_position
    )
    if distance > VEHICLE_PROXIMITY_THRESHOLD:
        return None

    # Find occupant candidates
    candidates = vehicle_agent.occupants

    # Evidence evaluation
    if len(candidates) == 1:
        return AUTO_MERGE(candidates[0].agent_id, person_track)

    # Multiple candidates - use face match
    for candidate in candidates:
        if person_track.has_face:
            similarity = face_recognition.compare(
                person_track.best_face_crop,
                candidate.agent.face_library
            )
            if similarity > 0.75:
                return AUTO_MERGE(candidate.agent_id, person_track)

    # No face match - check process of elimination
    unaccounted_candidates = [
        c for c in candidates
        if not c.agent.has_conflicting_track(person_track.start_time)
    ]

    if len(unaccounted_candidates) == 1:
        return MERGE_WITH_UNCERTAINTY(unaccounted_candidates[0], person_track)

    # Multiple unaccounted - uncertain
    return MULTI_ASSIGNMENT(unaccounted_candidates, person_track)
```

### Process of Elimination
```python
def rule_out_vehicle_occupant(agent_id, timestamp):
    """
    Agent seen elsewhere - can't be in vehicle

    Cascades through all vehicles with this occupant
    """
    for vehicle in active_vehicles:
        for occupant in vehicle.occupants:
            if occupant.agent_id == agent_id:
                # Remove from occupant list
                vehicle.occupants.remove(occupant)

                # Cascade: if only one occupant remains, increase confidence
                if len(vehicle.occupants) == 1:
                    vehicle.occupants[0].validated = True
```

## Related Documents
- **Source:** AGENT_TRACKING_AND_MERGE.md (deprecated)
- **L3_Merging:** Non-technical approach
- **L4_Complexity:** Where this gets hard
