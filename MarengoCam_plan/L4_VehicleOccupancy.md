# Level 4 - Vehicle Occupancy Tracking

## Purpose
Specification for tracking people entering and exiting vehicles. Enables cross-camera merging when agents use vehicles to move between locations.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L4_PortalTransitions.md](L4_PortalTransitions.md)

**Implemented in:** L14_VehicleTracker.py

---

## Concept

**Challenge:** Person becomes occluded when inside vehicle

**Solution:** Track vehicle as separate agent, maintain occupancy list

**Key question:** "Who is in the vehicle during this time window?"

---

## Vehicle Agent Model

```python
VehicleAgent(
    agent_id: str,  # "Vehicle_A"
    occupants: List[str],  # [Agent_A, Agent_B] currently inside

    # Entry/exit events
    entry_events: List[OccupancyEvent],
    exit_events: List[OccupancyEvent],

    # Visual characteristics
    color: str,  # "silver"
    license_plate: Optional[str],
)
```

---

## Occupancy Events

**Entry event:**
```python
OccupancyEvent(
    event_type: "entry",
    person_id: str,
    vehicle_id: str,
    timestamp: float,
    confidence: float,  # 0.0-1.0
)
```

**Triggers:**
- Person track ends within 2m of vehicle
- Vehicle door opens (if detectable)
- Vehicle starts moving shortly after

**Exit event:** Same structure, `event_type: "exit"`

---

## Occupancy Resolution

**Query: Who is in vehicle during time window?**

```python
def get_vehicle_occupancy(vehicle_id, start_time, end_time):
    # Start with all who entered before end_time
    entered = set(event.person_id for event in vehicle.entry_events
                  if event.timestamp < end_time)

    # Remove who exited before start_time
    exited = set(event.person_id for event in vehicle.exit_events
                 if event.timestamp < start_time)

    # Remove who were seen elsewhere during window (alibi)
    elsewhere = set(agent_id for agent_id in entered
                    if agent_seen_elsewhere(agent_id, start_time, end_time))

    return list(entered - exited - elsewhere)
```

---

## Process of Elimination

**Single occupant:**
```python
occupants = get_vehicle_occupancy("Vehicle_A", T1, T2)

if len(occupants) == 1:
    track_exit.global_agent_id = occupants[0]
    return "AUTO_MERGE"  # Definitive
```

**Multiple occupants:**
```python
if len(occupants) > 1:
    track_exit.identity_candidates = set(occupants)

    # Rely on face match or Re-ID
    if face_match(track_exit, Agent_A) >= 0.75:
        track_exit.global_agent_id = Agent_A
        # Process of elimination: other exit must be Agent_B
    else:
        return "QUEUE_FOR_REVIEW"
```

---

## Vehicle Configuration

```yaml
vehicles:
  - id: "Car_A"
    name: "Family Car"
    color: "silver"
    license_plate: "ABC123"
    max_capacity: 5

portals:
  - id: "car_driveway"
    type: "vehicle_entry"
    metadata:
      vehicle_id: "Car_A"
```

---

## Edge Cases

**Vehicle off-camera:** Occupancy confidence drops (unknown passengers may have been picked up)

**Driver vs passenger:** Don't distinguish (both are occupants)

**Vehicle in garage:** Occupants could have exited inside garage (no visibility)

---

## Metrics

**Single occupant accuracy:** Target > 95%

**Multiple occupant disambiguation:** Target > 70%

**Entry/exit detection rate:** Target > 90%

---

## Related Documents

**Tactical:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_Configuration.md](L3_Configuration.md)

**Other L4:** L4_PortalTransitions.md (vehicle portals), L4_AlibiCheck.md (process of elimination)

**Algorithms:** L12_VehicleOccupancy.md (detailed logic)

**Implementation:** L14_VehicleTracker.py, L14_OccupancyResolver.py
