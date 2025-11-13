# Level 3 - Cross-Camera Merging

## Purpose
How we merge track segments across cameras to build agent timelines.

## Merge Candidates

### Finding Candidates
**Periodic matching** (e.g., every 30 seconds):
- Compare recent track segments across all cameras
- Time window: tracks within ~60s of each other
- Portal configuration narrows candidates (user-defined connections)

### Portal Configuration
**User manually defines camera connections:**
- Which cameras connect (door, gate, driveway)
- Expected transition locations (cells/zones on each camera)
- Typical timing (optional, system learns this)

**System uses portals to:**
- Filter candidates (only check connected cameras)
- Note: Portal timing NOT used (variance too high, see L2D_Strategy)

## Evidence Evaluation

### Evidence Types

**Strong Evidence:**
- Face match (known person from library)
- Portal crossing (track ends at portal, next starts at corresponding portal)
- Only candidate (no other plausible options)
- Time/space exact fit

**Weak Evidence:**
- Spatial proximity (nearby locations)
- Timing fits (plausible travel time)
- Visual similarity (clothing, size)

### Decision Hierarchy

**1. Face Recognition** (fastest, most definitive)
- Match against face library
- Known person → auto-merge
- Unknown/no face → continue to next step

**2. Time/Space Gating** (eliminate impossible)
- Too fast: teleportation (reject)
- Too slow: implausible (reject)
- Candidates that pass: continue

**3. Portal Check** (user configuration)
- Track ends at portal on Camera A
- Next track starts at corresponding portal on Camera B
- Spatial correlation (not timing) → candidate

**4. LLM Adjudication** (expensive, ambiguous cases)
- Present image panels from both tracks
- Ask: "Same person/vehicle?"
- Forced-choice: YES/NO/CANNOT

**5. Human Validation** (final fallback)
- Queue for manual review
- User confirms or rejects merge
- System learns from decision

## Handling Uncertainty

### Multi-Assignment
**Track may belong to multiple agents:**
- Two people walk through portal together
- Tracks split on other side
- System marks: "Track X possibly Agent A or Agent B" (no probabilities)

### Uncertainty Persistence
**Don't force wrong merges:**
- Uncertain assignment can persist indefinitely
- Timeline shows possibilities
- Later evidence may resolve (face visible, process of elimination)

### Process of Elimination
**Ruling out candidates increases confidence:**
- Agent A seen elsewhere at same time → rules out Track X = Agent A
- Only one candidate remains → confidence increases
- Cascading updates as new evidence appears

## Learning from Validations

### Face Library
**Builds over time:**
- Human/LLM validates merge → register face
- Next appearance: auto-merge via face match
- Known people require less LLM/human review

## Vehicle Association Tactics

### Entry Detection
**When person track ends near vehicle:**
- Temporal correlation: person disappears within ~5s of vehicle movement
- Spatial correlation: person last seen within vehicle proximity zone
- Agent state → `in_vehicle(V)`
- Vehicle state → `occupied_by([Agent_A])`

### Sporadic Interior Detections
**YOLO detects person through windows (inconsistent):**
- Don't require frame-to-frame continuity
- Each detection: attempt face match against occupant candidates
- Face match → strengthens assignment
- No face → uncertain, but presence noted

### Exit Detection
**When vehicle parks and person appears:**
- Vehicle stops → state changes to `parked`
- Person track starts near vehicle → merge candidate
- Evidence: timing + spatial proximity + (optional) face match
- If multiple occupants: process of elimination (who else unaccounted for?)

### Multi-Occupant Handling
**Multiple people enter vehicle:**
- Vehicle `occupied_by([A, possibly B, possibly C])`
- Sporadic detections may identify some (face visible through window)
- Exit assignment: use face match if available, otherwise uncertain
- Accept uncertainty: "Vehicle had 2 people, 1 exited (possibly A or B)"

### Vehicle Portal Transitions
**Garage doors:**
- Vehicle enters garage → state: `in_structure`
- Vehicle exits garage → state: `moving`
- Portal type: `garage_door` (user-configured)

**Property exits:**
- Vehicle leaves via driveway → state: `offscreen`
- Portal type: `property_exit`
- Occupants remain `in_vehicle` (timeline: "left property in Vehicle V")

### Person-in-Vehicle Merge Exemption
**Critical rule:**
- Person with `in_vehicle` state skipped during cross-camera person merging
- Person can't walk camera-to-camera while in car
- Vehicle track is proxy for person movement
- Only look for person merges when person exits vehicle (state → `visible`)

## Related Documents
- **L2 (Strategy):** Why evidence hierarchy and uncertainty handling
- **L4 (Complexity):** Where merging gets complicated
- **L11_Merging:** Technical specs (algorithms, data structures)
