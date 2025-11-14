# Level 12 - Vehicle Occupancy Resolution
## Tracking People in and out of Vehicles

## Purpose

When multiple people enter or exit a vehicle, the system must determine:
- Which person is driving?
- Who is inside the vehicle?
- Who just exited?
- Where is each occupant now?

This document specifies how **process of elimination** applies to vehicle-related tracking decisions.

## 1. Core Concepts

### Vehicle vs. Person Agent

- **Vehicle Agent:** A tracked vehicle (car, truck, etc.) with states:
  - `visible_parked` - stationary on camera
  - `visible_moving` - actively driving
  - `in_structure(garage_id)` - parked inside garage/covered structure
  - `unknown` - lost track
  - `exited` - left property

- **Person Agent:** A tracked person with states:
  - `visible` - walking on camera
  - `visible_in_vehicle(V)` - sporadic detections through windows while in vehicle V
  - `in_vehicle(V)` - inside vehicle V (fully occluded, inferred)
  - `in_structure(building_id)` - inside building (inferred)
  - `offscreen` - between cameras
  - `unknown` - lost track
  - `exited` - left property

### Vehicle State Management

```python
class VehicleAgent:
    """A tracked vehicle."""
    agent_id: str
    agent_type: str = "vehicle"
    state: str  # visible_moving, visible_parked, in_structure, etc.

    # Occupancy tracking
    occupants: List[str]  # agent_ids of people inside
    entry_events: List[OccupancyEvent]  # When people entered
    exit_events: List[OccupancyEvent]  # When people exited

    # Temporal tracking
    last_seen_time: float
    last_seen_camera: str
    last_parked_location: Optional[Location]

    def get_likely_occupants(self) -> List[str]:
        """
        Return list of people likely still in vehicle.
        Exclude people confirmed to have exited or been elsewhere.
        """
        still_inside = []
        for person_id in self.occupants:
            person = get_agent(person_id)
            if not person_confirmed_exited(person, self.agent_id):
                still_inside.append(person_id)
        return still_inside
```

## 2. Occupancy Tracking Phases

### Phase 1: Person Entry into Vehicle

**Observable evidence:**
- Person track ends near vehicle
- Vehicle track starts moving (or already moving)
- Temporal correlation (person disappears, vehicle moves)

**Possible interpretations:**
1. Person entered vehicle → `in_vehicle(vehicle_id)` state
2. Person went inside building instead → `in_structure(building_id)` state
3. Person walked to another camera → continue tracking

**Decision logic:**
```python
def evaluate_person_entry_into_vehicle(person_track: Track, vehicle: VehicleAgent) -> Decision:
    """
    When a person track ends near a vehicle, determine if they entered the vehicle.
    """
    # Spatial check
    dist = distance(person_track.exit_cell, vehicle.last_position)
    if dist > ENTRY_DISTANCE_THRESHOLD:
        return REJECTED, "too_far_from_vehicle"

    # Temporal check
    time_delta = vehicle.next_movement_time - person_track.end_time
    if time_delta > VEHICLE_ENTRY_WINDOW:
        return REJECTED, "vehicle_moved_too_late"

    # Vehicle movement check
    if vehicle.state == "visible_parked" and time_delta > VEHICLE_DELAYED_ENTRY_WINDOW:
        # Person could have walked elsewhere, not necessarily entered vehicle
        return AMBIGUOUS, "vehicle_stationary"

    # Check for portal
    if is_building_entrance(person_track.exit_cell):
        # Person might have entered building instead
        return AMBIGUOUS, "building_entrance_nearby"

    # Decision: Mark person as in_vehicle
    return ASSIGN_TO_VEHICLE, "spatial_temporal_correlation"
```

### Phase 2: Vehicle Movement

**What happens:**
- Vehicle moves across cameras
- Person inside vehicle is effectively invisible (sporadic interior detections only)
- Person agent state: `in_vehicle(vehicle_id)`

**Tracking strategy:**
- Don't attempt to merge person tracks at different cameras
- Use vehicle track as proxy for person movement
- Sporadic interior detections (face through window) confirm occupancy but don't create new tracks

```python
def handle_sporadic_interior_detection(
    interior_detection: Detection,
    vehicle: VehicleAgent
) -> None:
    """
    Handle sporadic face/person detection through vehicle windows.
    Use for identification, not for continuous tracking.
    """
    likely_occupants = vehicle.get_likely_occupants()

    if len(likely_occupants) == 0:
        # No known occupants (shouldn't happen if tracking working)
        return

    if len(likely_occupants) == 1:
        # Single occupant - easy identification
        occupant_id = likely_occupants[0]
        occupant = get_agent(occupant_id)

        # Face similarity check
        face_sim = compute_face_similarity(interior_detection.face, occupant.face_library)
        if face_sim > 0.75:
            # Confirmed occupant
            log_interior_detection(occupant_id, vehicle.agent_id)
        return

    # Multiple occupants - ambiguous
    candidates = []
    for occupant_id in likely_occupants:
        occupant = get_agent(occupant_id)
        face_sim = compute_face_similarity(interior_detection.face, occupant.face_library)
        if face_sim > 0.70:
            candidates.append((occupant_id, face_sim))

    if len(candidates) == 1:
        # High-confidence match to one occupant
        log_interior_detection(candidates[0][0], vehicle.agent_id)
    elif len(candidates) > 1:
        # Multiple matches - ambiguous, log all possibilities
        log_ambiguous_interior_detection(candidates, vehicle.agent_id)
```

### Phase 3: Vehicle Parks and Person Exits

**Observable evidence:**
- Vehicle parks (state becomes `visible_parked` or `in_structure`)
- Shortly after, a person track appears near vehicle
- Temporal correlation (vehicle parks, person appears)

**Problem:** Multiple people may have been in vehicle, only one exits. Which person is it?

**Candidates:** Anyone who entered this vehicle and hasn't been confirmed exiting elsewhere

**Decision logic:**
```python
def evaluate_person_exit_from_vehicle(
    new_person_track: Track,
    vehicle: VehicleAgent,
    vehicle_park_time: float
) -> Decision:
    """
    When a person track appears near a parked vehicle, determine if they exited from it.
    Apply process of elimination to identify which occupant exited.
    """
    # Spatial check
    dist = distance(new_person_track.entry_cell, vehicle.last_parked_location)
    if dist > EXIT_DISTANCE_THRESHOLD:
        return REJECTED, "too_far_from_vehicle"

    # Temporal check
    time_delta = new_person_track.start_time - vehicle_park_time
    if time_delta > VEHICLE_EXIT_WINDOW:
        return REJECTED, "too_long_after_parking"

    # PROCESS OF ELIMINATION: Which occupant could have exited?
    likely_occupants = vehicle.get_likely_occupants()

    if len(likely_occupants) == 0:
        # Vehicle was empty or all occupants already identified exiting
        return NEW_AGENT, "vehicle_was_empty"

    if len(likely_occupants) == 1:
        # Only one person in vehicle - must be them
        return AUTO_MERGE, f"only_occupant_{likely_occupants[0]}"

    # Multiple occupants - apply constraints
    candidates_after_elimination = apply_exit_constraints(
        new_person_track,
        likely_occupants
    )

    if len(candidates_after_elimination) == 1:
        # Deterministic via process of elimination
        return AUTO_MERGE, f"poe_single_candidate_{candidates_after_elimination[0]}"

    elif len(candidates_after_elimination) > 1:
        # Still ambiguous after constraints
        return AMBIGUOUS, f"multiple_candidates_{candidates_after_elimination}"

    else:
        # All candidates eliminated - new person?
        return AMBIGUOUS, "no_candidate_fit"
```

## 3. Constraint Application for Vehicle Exit

### Constraint 1: Temporal Bounds

**Rule:** If a person agent is confirmed to be somewhere else at time T, they can't exit vehicle at T.

```python
def temporal_constraint(
    exit_person_track: Track,
    candidate_occupant: str,
    vehicle_exit_time: float
) -> bool:
    """
    Returns: True if candidate is eliminated, False if they remain plausible.
    """
    person = get_agent(candidate_occupant)

    # Check if person confirmed visible elsewhere
    if person.last_seen_camera is not None:
        last_seen_time = person.last_seen_time
        last_seen_camera = person.last_seen_camera

        # If person was tracked after vehicle park time, they can't be in vehicle
        if last_seen_time > vehicle_exit_time:
            return True  # ELIMINATED

        # Check plausibility of return to vehicle
        if last_seen_time < vehicle_exit_time:
            # Person was elsewhere but exited camera. Could have returned to vehicle?
            time_gap = vehicle_exit_time - last_seen_time
            vehicle_current_location = exit_person_track.camera_id

            # If person's last location is far from vehicle, unlikely they returned
            distance_to_vehicle = distance_between_cameras(
                last_seen_camera,
                vehicle_current_location
            )

            if time_gap < distance_to_vehicle / WALKING_SPEED:
                # Not enough time to walk back and get picked up again
                return True  # ELIMINATED

    return False  # Still plausible
```

### Constraint 2: Vehicle State

**Rule:** If vehicle is in_structure (garage), occupant likely still there, not exiting.

```python
def vehicle_state_constraint(
    vehicle: VehicleAgent,
    exit_time: float,
    vehicle_state_at_time: str
) -> bool:
    """
    Returns: True if candidate eliminated by vehicle state.
    """
    if vehicle_state_at_time == "in_structure":
        # Vehicle is in garage/structure, unlikely person exited to visible camera
        # (they'd have to exit building first)
        return True  # ELIMINATE: too restrictive, modify logic

    # Actually, person could have exited to interior of garage
    # Let evidence engine determine plausibility
    return False
```

### Constraint 3: Known Exit Behavior

**Rule:** If person is tracked exiting at a different location at same time, can't be exiting vehicle.

```python
def conflicting_location_constraint(
    exit_person_track: Track,
    candidate_occupant: str,
    vehicle: VehicleAgent
) -> bool:
    """
    Check if candidate has conflicting track at same time.
    """
    person = get_agent(candidate_occupant)

    # Look for tracks at different cameras around same time
    for track in person.tracks:
        if abs(track.start_time - exit_person_track.start_time) < 2.0:
            # Track starts at nearly same time as vehicle exit
            # Can't be same person at two places
            return True  # ELIMINATED

    return False
```

## 4. Multi-Occupant Disambiguation

**Scenario:** Two people entered vehicle, vehicle moved, vehicle parks, one person exits.

```
T=10s: Person-A enters vehicle (track ends)
T=11s: Person-B enters vehicle (track ends)
Vehicle occupants: [Person-A, Person-B]

T=15s: Vehicle parks at garage

T=16s: Person appears on camera near garage exit
       → Could be Person-A or Person-B

Evidence for disambiguation:
1. Face similarity: Does new track look like Person-A or Person-B?
2. Temporal: Was Person-A or Person-B tracked elsewhere?
3. Vehicle camera: Did interior detection match either?
4. Process of elimination: Which other occupants can be ruled out?

```

**Implementation:**

```python
def disambiguate_vehicle_exit(
    exit_person_track: Track,
    vehicle: VehicleAgent,
    all_evidence: EvidenceReport
) -> List[Tuple[str, float]]:
    """
    Given a person exiting vehicle with multiple possible occupants,
    return candidates ranked by likelihood.

    Returns: List of (occupant_id, confidence_score)
    """
    likely_occupants = vehicle.get_likely_occupants()
    candidate_scores = {}

    for occupant_id in likely_occupants:
        score = 0.0

        # EVIDENCE 1: Face similarity
        if all_evidence.face_similarity[occupant_id] > 0.7:
            score += 40  # Strong evidence

        # EVIDENCE 2: Temporal constraints
        temporal_violations = temporal_constraint(
            exit_person_track,
            occupant_id,
            vehicle.park_time
        )
        if temporal_violations:
            score -= 100  # Eliminate

        # EVIDENCE 3: Interior detections
        if occupant_id in all_evidence.interior_detections_matching:
            score += 20

        # EVIDENCE 4: Process of elimination
        if len(likely_occupants) == 1:
            score += 50  # Only option
        elif len(likely_occupants) == 2 and temporal_constraint(...):
            score += 30  # Other eliminated

        candidate_scores[occupant_id] = score

    # Sort by confidence
    ranked = sorted(
        candidate_scores.items(),
        key=lambda x: x[1],
        reverse=True
    )

    return ranked
```

## 5. Handling Unresolved Vehicle Exits

**Problem:** Vehicle has multiple occupants, one exits, but we can't determine who.

**Solution:** Mark exit person as ambiguous multi-assignment.

```python
def handle_ambiguous_vehicle_exit(
    exit_person_track: Track,
    likely_occupants: List[str]
):
    """
    When vehicle exit is ambiguous, mark track with candidate list.
    Let system resolve through future evidence (face match, timeline).
    """
    exit_person_track.agent_id = None
    exit_person_track.candidate_agents = likely_occupants
    exit_person_track.elimination_log = {
        occupant: "vehicle_exit_ambiguous"
        for occupant in likely_occupants
    }

    # Timeline: "Unknown person exited vehicle [possibly Person-A or Person-B]"
```

## 6. Vehicle Entry and Exit Event Logging

```python
class OccupancyEvent:
    """Record of a person entering/exiting vehicle."""
    event_type: str  # "entry" or "exit"
    person_agent_id: str
    vehicle_agent_id: str
    event_time: float
    location: Location  # Where entry/exit occurred
    confidence: float  # 0.0-1.0
    evidence_reasons: List[str]  # Why we think this happened

def log_occupancy_event(
    event_type: str,
    person: str,
    vehicle: str,
    time: float,
    confidence: float,
    reasons: List[str]
):
    """
    Log person entry/exit event on vehicle.
    Used for timeline reconstruction and forensic audit.
    """
    event = OccupancyEvent(
        event_type=event_type,
        person_agent_id=person,
        vehicle_agent_id=vehicle,
        event_time=time,
        confidence=confidence,
        evidence_reasons=reasons
    )
    vehicle.entry_events.append(event)  # or exit_events

    # Update person state
    person_agent = get_agent(person)
    if event_type == "entry":
        person_agent.state = f"in_vehicle({vehicle})"
    elif event_type == "exit":
        person_agent.state = "visible"

    # Timeline note
    if confidence > 0.9:
        note = f"Person {person_agent.name} {'entered' if event_type == 'entry' else 'exited'} {vehicle_agent.label}"
    else:
        note = f"Person {person_agent.name} possibly {'entered' if event_type == 'entry' else 'exited'} {vehicle_agent.label}"

    timeline.add_event(note, time, person_agent_id)
```

## 7. Related Documents

- **L12_ProcessOfElimination:** System-wide constraint-based disambiguation pattern (applies to vehicle exit)
- **L12_Merging.md:** Portal-based merging logic and alternative source detection
- **L2_Strategy.md:** Strategy for vehicle association (sections on observable evidence, agent states)
- **L13_TrackStateManager:** Track data structures and agent promotion
- **L13_TimelineReconstruction:** Converting occupancy events to narrative timeline
