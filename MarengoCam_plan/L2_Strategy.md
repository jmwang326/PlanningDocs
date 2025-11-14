# Level 2 - Strategy

## Purpose
Non-technical description of **what** we're doing and **why** this approach solves the problem.

## Strategy Overview

### Filter Noise First
**What:** Only promote sustained movement to "real activity"
**Why:** Trees/shadows/brief glitches create false alerts
**Approach:** Object must move continuously (not just detected once)

### Track Within Cameras
**What:** Follow objects frame-to-frame within single camera
**Why:** YOLO11 provides tracking, reduces per-frame detections to persistent tracks
**Approach:**
- YOLO11 links detections across frames to create continuous tracks
- Movement is required - static detections don't generate tracks
- Temporal continuity built through chunked processing with overlap zones
- YOLO provides local tracking IDs within chunks, system links across chunks

### Merge Across Cameras
**What:** Link track segments that belong to same agent
**Why:** Timeline requires knowing when Track A (Camera 1) = Track B (Camera 2)
**Approach:** Evidence-based matching (face, time/space, portals)

### Evidence Hierarchy
**What:** Use strongest evidence first, fallback to weaker evidence
**Why:** Face match is definitive, grid overlap eliminates ambiguity, LLM is expensive

**Hierarchy:**
1. **Face recognition** (known people auto-merge)
2. **High-Confidence Time/Space Match** (e.g., validated grid overlap with a single candidate)
3. **Plausible Time/Space Match** (e.g., learned grid adjacency, then evaluated with other evidence)
4. **LLM / Human Adjudication** (for ambiguous cases)

### Accept Uncertainty
**What:** Some tracks can't be assigned definitively
**Why:** Evidence may be missing (face occluded, timing ambiguous, multiple candidates)

**Approach:**
- Mark tracks as "possibly Agent A or Agent B"
- Uncertainty persists until resolved by later evidence
- System operates with partial timeline
- Don't force wrong merge to avoid uncertainty

### Grid-Based Spatial Learning
**What:** A single `6x6` inter-camera grid learns which camera regions connect. This is the sole system for inter-camera learning, superseding older, more complex multi-grid approaches.
**Why:** Distinguishes overlap (same view) from portals (movement between cameras).

**Approach:**
- Bootstrap from historical validated merges
- Grid learns typical transition times between cell pairs
- Overlap zones (typical_time ≈ 0): auto-merge same agent
- Portal zones (typical_time 1-15s): normal evidence evaluation
- No connection (no data): reject or manual portal config

### Learn from Corrections
**What:** Human/LLM validations improve future decisions
**Why:** System learns what "same person" looks like over time

**Approach:**
- Face library builds from validated merges
- Grid learns transition patterns from validated cross-camera merges
- Face recognition improves as library grows

## Strategy for Noise Filtering

### Multi-Layer Filtering
**Layer 1 - Motion gate (Standby cameras):**
- Fast pixel-diff check before YOLO inference
- Filters static frames (trees, shadows, parked cars)
- Only frames with motion delta proceed to YOLO

**Layer 2 - Movement requirement:**
- YOLO detections must show sustained movement to create tracks
- **Static objects DO NOT generate active tracks** - stationary agents exist but can't be merged (no track to merge)
- Example: Parked car is an agent with state `visible_parked`, but has no active track. When car starts moving, track created with state `visible_moving`, enabling cross-camera merging
- Camera promoted to Active only when moving track exists, not from static detections

**Layer 3 - Track quality:**
- Duration: must persist across frames (≥3s threshold)
- Confidence: detector confidence above threshold
- Object type: person, vehicle, animal (not "unknown")

**What gets filtered:**
- Static frames (motion gate)
- Brief detections (< threshold duration)
- Low confidence objects
- Non-moving detections (parked car, tree)
- Detector glitches

### Timeline Construction
**Goal:** "What happened today?" answered by agent timeline, not camera-by-camera logs

**Approach:**
- Merge tracks into agent journeys
- Show agent movement across property
- Filter out noise (never promoted to timeline)

## Strategy for Uncertainty

### Types of Uncertainty

**Identity Uncertainty:**
- Track might belong to Agent A or Agent B
- Example: Two people walk through portal together, tracks split on other side

**Location Uncertainty:**
- Agent exists but current location unknown
- Example: Person enters building, not visible for 10 minutes
- Example: Person enters vehicle, vehicle moves across cameras

**Assignment Uncertainty:**
- Can't determine if Track X belongs to existing agent or is new agent
- Example: Face occluded, timing fits multiple candidates

**Vehicle Association Uncertainty:**
- Person enters vehicle, which person exits?
- Example: Two people enter car, car moves, one person exits (driver or passenger?)

### Handling Uncertainty

**Don't force decisions:**
- Wrong merge worse than uncertain merge
- Uncertainty can persist indefinitely
- Later evidence may resolve (person exits building, face visible)

**Multi-assignment possible:**
- Track marked as "possibly Agent A or Agent B" (no probabilities)
- Timeline shows all possibilities
- User can manually resolve if needed

**Process of elimination:**
- Agent A seen elsewhere → rules out Track X belonging to Agent A
- Cascades: ruling out candidates increases confidence in remaining

## Strategy for Vehicle Association

### Observable Evidence
**What we can detect:**
- Person track ends near vehicle (temporal/spatial correlation)
- Vehicle track starts moving
- Sporadic person detections inside vehicle (YOLO through windows)
- Vehicle parks, person track starts nearby (exit correlation)

**What we use:**
- Entry correlation: person disappears + vehicle moves → `in_vehicle` state
- Sporadic interior detections: confirms occupancy, may provide face match
- Exit correlation: vehicle parks + person appears → merge candidate
- Process of elimination: other agents accounted for → increases confidence

### Agent States

**Person agent states:**
- `visible` - actively tracked walking/moving on camera
- `visible_in_vehicle(V)` - sporadic detections through vehicle windows (fragmented tracks, special handling)
- `offscreen` - between cameras (expected portal transition, time-bounded)
- `in_structure(XYZ)` - entered building/garage via portal (inferred from outdoor cameras)
- `in_vehicle(V)` - inside vehicle, not visible (fully occluded)
- `unknown` - lost track, location unclear
- `exited` - crossed property exit portal, confirmed departure

**Vehicle agent states:**
- `visible_parked` - tracked on camera, stationary (boring, camera stays Armed)
- `visible_moving` - tracked on camera, in motion (interesting, Active tracking)
- `offscreen` - between cameras (expected portal transition, time-bounded)
- `in_structure(XYZ)` - entered garage via portal (inferred from outdoor cameras)
- `unknown` - lost track, location unclear
- `exited` - crossed property exit portal, confirmed departure
- `occupied_by([A, possibly B])` - list of person agents inside (with uncertainty)

### Strategy
**Don't force person→vehicle assignments:**
- Multiple people near vehicle → multi-assignment (`possibly A or B in vehicle`)
- Sporadic interior detection → confirmation (face match strengthens assignment)
- Exit uncertainty → process of elimination (who else is unaccounted for?)

**Accept gaps:**
- Person in vehicle has no continuous track (sporadic detections only)
- Timeline shows: "Agent A entered vehicle, vehicle moved, Agent A exited"
- Gaps acceptable (forensic reconstruction, not real-time tracking)

**Person-in-vehicle exemptions:**
- Person with `in_vehicle(V)` or `visible_in_vehicle(V)` state exempt from normal cross-camera merge rules
- Can't walk camera-to-camera while in car (vehicle does the moving)
- Sporadic `visible_in_vehicle` detections used for face matching, not continuous tracking
- Person track effectively "paused" until vehicle parks and person exits
- Vehicle track becomes proxy for person movement

### System Lifecycle
The system's lifecycle is defined by an initial **Bootstrapping** phase (see `L5_Bootstrap.md`) followed by a phased live deployment (see `L6_Deployment.md`).

## Related Documents
- **L1 (Mission):** Why we're solving this problem (core values, goals)
- **L3 (Tactics):** How components implement this strategy
- **L4 (Complexity):** Where this strategy gets complicated
- **L10 (Nomenclature):** Terminology definitions
