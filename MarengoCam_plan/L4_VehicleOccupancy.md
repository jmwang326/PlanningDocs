# Level 4 - Vehicle Occupancy Tracking

## Purpose
Detailed specification for tracking people entering and exiting vehicles. This concept enables cross-camera merging when agents use vehicles to move between locations.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L4_PortalTransitions.md](L4_PortalTransitions.md)

**Implemented in:** L14_VehicleTracker.py

---

## Core Concept

### Why Track Vehicle Occupancy

**Scenarios requiring vehicle tracking:**
1. Person enters car in driveway, car drives away
2. Car returns hours later, person exits
3. Multiple people enter car (driver + passengers)
4. Car moves between cameras, occupants exit at different locations

**Challenge:** Person becomes occluded when inside vehicle

**Solution:** Track vehicle as separate agent, maintain occupancy list

---

## Vehicle Agent Model

### Data Structure

```python
VehicleAgent(
    agent_id: str,  # "Vehicle_A"
    agent_type: str = "vehicle",
    state: str,  # "visible_parked", "visible_moving", "in_structure", "unknown"

    # Occupancy tracking
    occupants: List[str],  # [Agent_A, Agent_B] currently inside
    max_capacity: int = 8,  # Maximum occupants (default 8 for car)

    # Entry/exit events
    entry_events: List[OccupancyEvent],
    exit_events: List[OccupancyEvent],

    # Temporal tracking
    last_seen_time: float,
    last_seen_camera: str,
    last_parked_location: Optional[(camera_id, cell)],

    # Visual characteristics (for Re-ID)
    make_model: Optional[str],  # "Honda Civic"
    color: str,  # "silver"
    license_plate: Optional[str],  # If readable
)
```

### Vehicle States

**visible_parked:**
- Vehicle stationary, detected by YOLO
- Doors may open/close (entry/exit events)

**visible_moving:**
- Vehicle in motion
- No entry/exit allowed while moving

**in_structure:**
- Vehicle inside garage (not visible)
- Inferred from garage door portal

**unknown:**
- Lost track (vehicle left property or moved between cameras)

---

## Occupancy Events

### Entry Event

**Observable evidence:**
```python
OccupancyEvent(
    event_type: "entry",
    person_id: str,  # Agent_A
    vehicle_id: str,  # Vehicle_A
    timestamp: float,  # When entry observed
    camera_id: str,  # Where observed
    confidence: float,  # 0.0-1.0 (certainty of entry)
    evidence: str,  # "person_track_ended_at_vehicle"
)
```

**Triggers:**
- Person track ends within 2 meters of vehicle
- Vehicle door opens (if detectable)
- Vehicle starts moving shortly after

**Confidence levels:**
- **High (> 0.8):** Person clearly enters vehicle, door visible
- **Medium (0.5-0.8):** Person approaches vehicle, disappears
- **Low (< 0.5):** Person near vehicle, ambiguous (could have entered building)

---

### Exit Event

**Observable evidence:**
```python
OccupancyEvent(
    event_type: "exit",
    person_id: str,  # Agent_A
    vehicle_id: str,  # Vehicle_A
    timestamp: float,  # When exit observed
    camera_id: str,  # Where observed
    confidence: float,  # 0.0-1.0
    evidence: str,  # "person_track_started_at_vehicle"
)
```

**Triggers:**
- Person track starts near vehicle
- Vehicle door opens (if detectable)
- Vehicle was recently parked (< 30s)

---

## Occupancy Resolution

### Query: Who Is in Vehicle?

```python
def get_vehicle_occupancy(vehicle_id, start_time, end_time):
    """
    Return list of people likely in vehicle during time window.
    """
    vehicle = get_vehicle_agent(vehicle_id)

    # Start with all entry events before end_time
    entered = set(
        event.person_id
        for event in vehicle.entry_events
        if event.timestamp < end_time
    )

    # Remove people who exited before start_time
    exited = set(
        event.person_id
        for event in vehicle.exit_events
        if event.timestamp < start_time
    )

    # People confirmed elsewhere during window (alibi)
    confirmed_elsewhere = set(
        agent_id
        for agent_id in entered
        if agent_seen_elsewhere(agent_id, start_time, end_time)
    )

    # Remaining = entered - exited - confirmed_elsewhere
    occupants = entered - exited - confirmed_elsewhere

    return list(occupants)
```

**Example:**
```
Vehicle_A entry events:
  T=08:00 - Agent_A enters
  T=08:02 - Agent_B enters

Vehicle_A exit events:
  T=08:45 - Agent_A exits (at grocery store)

Query: get_vehicle_occupancy("Vehicle_A", T=08:50, T=09:00)
Result: [Agent_B]  (Agent_A exited, only Agent_B remains)
```

---

## Entry/Exit Detection

### Detecting Entry

**Spatial correlation:**
```python
def detect_vehicle_entry(person_track, vehicle):
    # Person track ended near vehicle
    distance = calc_distance(person_track.exit_cell, vehicle.location)

    if distance < 2.0 and vehicle.state == "visible_parked":
        # Person approached vehicle, disappeared
        confidence = 0.7

        # Increase confidence if vehicle moved shortly after
        if vehicle.started_moving_within(person_track.end_time, 30):
            confidence = 0.9

        # Create entry event
        vehicle.entry_events.append(
            OccupancyEvent(
                event_type="entry",
                person_id=person_track.global_agent_id,
                vehicle_id=vehicle.agent_id,
                timestamp=person_track.end_time,
                confidence=confidence
            )
        )

        vehicle.occupants.append(person_track.global_agent_id)
```

---

### Detecting Exit

**Spatial correlation:**
```python
def detect_vehicle_exit(person_track, vehicle):
    # Person track started near vehicle
    distance = calc_distance(person_track.entry_cell, vehicle.location)

    if distance < 2.0 and vehicle.state == "visible_parked":
        # Person appeared near vehicle
        confidence = 0.7

        # Increase confidence if vehicle recently arrived
        if vehicle.parked_within(person_track.start_time, 60):
            confidence = 0.9

        # Create exit event
        vehicle.exit_events.append(
            OccupancyEvent(
                event_type="exit",
                person_id=person_track.global_agent_id,
                vehicle_id=vehicle.agent_id,
                timestamp=person_track.start_time,
                confidence=confidence
            )
        )

        vehicle.occupants.remove(person_track.global_agent_id)
```

---

## Process of Elimination (Vehicles)

### Single Occupant Case

**Scenario:**
```
T=08:00 - Agent_A enters Vehicle_A (only occupant)
T=08:30 - Vehicle_A drives to Store camera
T=08:35 - Person exits Vehicle_A (Track_X, identity unknown)

Question: Who is Track_X?
Answer: Must be Agent_A (only occupant)
```

**Logic:**
```python
occupants = get_vehicle_occupancy("Vehicle_A", T=08:30, T=08:35)
# Result: [Agent_A]

if len(occupants) == 1:
    # Only one person in vehicle, must be them
    track_X.global_agent_id = occupants[0]
    return "AUTO_MERGE"  # Definitive
```

---

### Multiple Occupant Case

**Scenario:**
```
T=08:00 - Agent_A enters Vehicle_A (driver)
T=08:02 - Agent_B enters Vehicle_A (passenger)
T=08:30 - Vehicle_A drives to Store camera
T=08:35 - Person exits Vehicle_A (Track_X, identity unknown)
T=08:40 - Person exits Vehicle_A (Track_Y, identity unknown)

Question: Which exit = which person?
```

**Logic:**
```python
occupants = get_vehicle_occupancy("Vehicle_A", T=08:30, T=08:40)
# Result: [Agent_A, Agent_B]

# Multi-assignment (both tracks uncertain)
track_X.identity_candidates = {Agent_A, Agent_B}
track_Y.identity_candidates = {Agent_A, Agent_B}

# Rely on face match or Re-ID to disambiguate
if face_match(track_X, Agent_A) >= 0.75:
    track_X.global_agent_id = Agent_A
    # Process of elimination: Track_Y must be Agent_B
    track_Y.global_agent_id = Agent_B
else:
    # Queue both for review
    return "QUEUE_FOR_REVIEW"
```

---

## Vehicle Tracking

### Vehicle Detection

**YOLO classes:** `car`, `truck`, `motorcycle`

**Vehicle Re-ID features:**
- Color (primary discriminator)
- Make/model (if detectable)
- License plate (if readable, high confidence)

**Vehicle persistence:**
```python
VehicleTrack(
    local_id: int,  # YOLO tracking ID
    camera_id: str,
    chunk_id: int,
    start_time: float,
    end_time: float,

    # Vehicle-specific
    vehicle_agent_id: Optional[str],  # Linked to VehicleAgent
    color: str,  # "silver", "black", "white"
    vehicle_class: str,  # "car", "truck"
    is_moving: bool,  # Stationary vs moving
)
```

---

### Vehicle Merging

**Cross-camera vehicle merging:**
- **Primary:** License plate match (if readable)
- **Secondary:** Color + make/model + timing
- **Fallback:** Human review

**Challenge:** Many vehicles look similar (silver sedans)

**Mitigation:**
- Focus on resident vehicles (configured in system)
- License plate recognition (if available)
- Accept uncertainty for visitor vehicles

---

## Configuration

### Registered Vehicles

**In `marengo.yaml`:**
```yaml
vehicles:
  - id: "Car_A"
    name: "Family Car"
    make_model: "Honda Civic"
    color: "silver"
    license_plate: "ABC123"
    max_capacity: 5
    typical_driver: "Agent_A"  # Hint, not enforced

  - id: "Truck_B"
    name: "Work Truck"
    make_model: "Ford F-150"
    color: "black"
    license_plate: "XYZ789"
    max_capacity: 3
```

**Usage:**
- Pre-configured vehicles are easier to track (known color/make)
- Typical driver is hint (not enforced by system)

---

### Vehicle Entry Portals

**Link portals to vehicles:**
```yaml
portals:
  - id: "car_driveway"
    name: "Car in Driveway"
    camera_id: "Driveway"
    cells: [[1, 1], [1, 2], [2, 1], [2, 2]]
    type: "vehicle_entry"
    metadata:
      vehicle_id: "Car_A"  # Link to Vehicle_A
```

**See:** L4_PortalTransitions.md for portal configuration

---

## Edge Cases

### Case 1: Vehicle Moves Off-Camera

**Problem:** Vehicle drives off property, returns later

**Solution:**
```python
if vehicle.last_seen_time < (now - 3600):
    # Vehicle been gone > 1 hour
    # Occupancy may have changed (unknown passengers picked up)
    vehicle.occupants_confidence = "low"
    # Rely on exit events for disambiguation
```

---

### Case 2: Driver vs Passenger

**Problem:** Who is driving vs who is passenger?

**Current approach:** Don't distinguish (both are occupants)

**Future enhancement:** Track driver seat (if detectable through windshield)

---

### Case 3: Vehicle in Garage

**Problem:** Vehicle enters garage, becomes concealed

**Solution:**
```python
if vehicle.at_portal == "garage_door":
    vehicle.state = "in_structure"
    vehicle.occupants_confidence = "medium"
    # Occupants could have exited inside garage (no visibility)
```

**Uncertainty:** People may exit vehicle inside garage (not visible)

**Mitigation:** Accept uncertainty, queue for review if critical

---

## Vehicle Occupancy Metrics

### Performance Indicators

**Single occupant accuracy:**
- Target: > 95% (easy case)
- Metric: % of single-occupant exits correctly identified

**Multiple occupant disambiguation rate:**
- Target: > 70% (harder case)
- Metric: % of multi-occupant exits correctly disambiguated

**Entry/exit detection rate:**
- Target: > 90%
- Metric: % of actual entries/exits detected (vs ground truth)

---

## Related Documents

### Tactical Context
- **L3_IdentityResolution.md** - Vehicle occupancy in merging logic
- **L3_Configuration.md** - Vehicle and portal configuration

### Other L4 Concepts
- **L4_PortalTransitions.md** - Vehicle portals (type: vehicle_entry)
- **L4_AlibiCheck.md** - Process of elimination for occupants

### Algorithms
- **L12_VehicleOccupancy.md** - Detailed occupancy resolution logic

### Implementation
- **L14_VehicleTracker.py** - Vehicle detection, occupancy tracking
- **L14_OccupancyResolver.py** - Entry/exit event processing
