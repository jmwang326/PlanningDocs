# Level 2 - Strategy

## Purpose
Non-technical description of **what** we're doing and **why** this approach solves the problem.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for the technical architecture (tracks + handlers model).

---

## Core Strategy: Manage Video Clips (Tracks)

**The fundamental insight:** Multi-camera surveillance is a **clip management problem**.

Every 10 seconds, YOLO processes 12-second video chunks and produces observations (LocalTracks). Our job is to:
1. **Link clips** from the same person across time (same camera)
2. **Identify clips** as belonging to specific people (cross-camera)
3. **Handle uncertainty** when we can't be sure

Everything else - timelines, alerts, forensics - is just querying and displaying these clips.

---

## Strategy Part 1: Create Tracks (ChunkProcessor)

### Filter Noise First
**What:** Only create tracks for sustained, moving activity.

**Why:** Trees, shadows, brief glitches create false detections. We need to filter these before creating tracks.

**Approach:**
- **Motion gate:** Fast pixel-diff check before running YOLO (filters static frames)
- **Movement requirement:** Detections must show sustained movement (≥3s threshold)
- **Confidence filter:** Only high-confidence detections (≥ threshold)

**What gets filtered out:**
- Static frames (motion gate)
- Brief detections (< 3s duration)
- Low confidence objects
- Stationary objects (parked car, tree)
- Detector glitches

**Result:** Only "real activity" becomes a LocalTrack.

---

## Strategy Part 2: Link Tracks Over Time (TemporalLinker)

### Track Within Camera
**What:** Connect LocalTracks from sequential chunks when they represent the same person.

**Why:** YOLO assigns new IDs each chunk. We need to link them to create continuous observation on one camera.

**Approach:**
- **Overlap zone:** Last 2s of chunk N overlaps first 2s of chunk N+1
- **Spatial continuity:** Match tracks by position in overlap zone
- **Appearance similarity:** If multiple candidates, use visual similarity
- **One-to-one matching:** Each track links to at most one track in next chunk

**Example:**
```
Chunk N:   YOLO_ID=3 (Gardener, T=0-12s)
Overlap:   T=10-12s, Gardener visible
Chunk N+1: YOLO_ID=5 (same person, T=10-22s, YOLO reassigned ID)

TemporalLinker connects:
  Chunk_N.track_3.next_track = (Chunk_N+1, track_5)
  Chunk_N+1.track_5.previous_track = (Chunk_N, track_3)
```

**Concurrent agents:**
- If multiple people in overlap zone, match by appearance + position
- Each person gets separate linked chain
- YOLO keeps them separate (different local_ids)

**Result:** Continuous timeline on one camera, even though YOLO IDs change.

---

## Strategy Part 3: Identify Tracks (IdentityResolver)

### Merge Across Cameras
**What:** Determine which global agent each LocalTrack belongs to.

**Why:** Timeline requires knowing when Track A (Camera 1) = Track B (Camera 2) = same person.

**Approach:** Evidence-based hierarchy (strongest evidence first).

---

### Evidence Hierarchy

**1. Face Recognition (Definitive)**
- CodeProject.AI face library
- Confidence ≥ 0.75 → auto-merge
- Most reliable evidence

**2. Grid Overlap (Definitive if single agent)**
- Travel time < 0.4s between cameras
- Same physical space visible in both cameras
- If only one agent present → must be same person → auto-merge

**3. Grid Transition + Alibi Check (Definitive if no alternatives)**
- Travel time fits learned 6x6 grid pattern
- **Alibi check:** Could entry agent have come from elsewhere?
- If NO other sources → process of elimination → auto-merge
- If YES other sources → ambiguous → queue for review

**4. LLM / Human Adjudication (Fallback)**
- Time/space plausible but ambiguous
- Multiple candidates fit the evidence
- Visual comparison panels + context → LLM/human decides

**5. Reject (Definitive)**
- Face mismatch
- Spatial/temporal impossibility
- Alibi check fails (seen elsewhere)

---

### Grid-Based Spatial Learning

**What:** Automatically learn minimum travel times between camera pairs at the 6x6 grid level.

**How it works:**
1. **Observation:** Person walks from Camera A to Camera B (validated merge)
2. **Measurement:** `travel_time = track_B.start_time - track_A.end_time`
3. **Mapping:** Map to grid cells: "exited Camera A at cell (5,2), entered Camera B at cell (0,3), took 1.5 seconds"
4. **Learning:** Store `grid[A→B][(5,2)→(0,3)] = 1.5 seconds` (keep minimum observed time)
5. **Future merging:** Similar exit/entry cells? Recognize as plausible transition

**Overlap zones (fastest merging):**
- Travel time < 0.4s → cameras see overlapping areas
- Grid cell marked as overlap (time ≈ 0)
- Future tracks in overlap cells with time ≈ 0 → **instant auto-merge** (must be same agent if only one visible)
- Key insight: **no time gap = same space = same agent**

**Adjacent camera transitions:**
- Travel time ≥ 0.4s → cameras see different areas
- Before auto-merging: **alibi check** (could agent have come from elsewhere?)
- If no alternatives → deterministic merge
- If alternatives exist → ambiguous → queue for review

**Why this matters:**
- Multiple people on property at same time
- Same grid cells used by different agents in sequence
- Alibi check prevents false merges

**Example:**
```
Person A exits front door at T=2s
Person B enters back camera at T=5s (3s later)

Grid says: front door → back camera = ~4s typical

Question: Is Person B actually Person A?

Alibi check: "Could any other camera's exit reach back camera at T=5s?"
- If NO: Must be Person A (deterministic)
- If YES: Ambiguous (need face/LLM)
```

**Edge noise handling:**
- Grid learns only from **validated merges** (single agent, high confidence)
- Excludes groups (multiple people together)
- Excludes ambiguous cases
- This filters boundary detection noise automatically

**Manual portals (for occluded spaces):**
- Doors, gates where agents enter buildings/vehicles
- User configures in YAML (system can't learn these from video)
- When track ends at portal → marks context flag: `at_portal = "garage_door"`
- Travel time doesn't apply (crossing boundary is what matters)

**Combined power:**
- Grid = automatic discovery of visible pathways (fast, data-driven)
- Manual portals = semantic knowledge of hidden spaces (required for timeline reconstruction)

---

### Process of Elimination

**What:** Rule out impossible candidates to narrow identity.

**Why:** When multiple agents on property, seeing one agent elsewhere rules them out as source for a new track.

**How it works:**
```
# 2 people exit garage
track_1.identity_candidates = {Agent_A, Agent_C}
track_2.identity_candidates = {Agent_A, Agent_C}

# Evidence: track_1 face match = Agent_A
track_1.global_agent_id = Agent_A
track_1.identity_candidates.clear()

# Elimination: Agent_A can't be in track_2
track_2.identity_candidates.remove(Agent_A)
# Only Agent_C remains
track_2.global_agent_id = Agent_C  # Resolved!
```

**Cascades:**
- Ruling out candidates increases confidence in remaining
- Eventually resolves to single agent (or stays ambiguous)
- Never forces decision if insufficient evidence

---

## Strategy Part 4: Resolve Uncertainty (EvidenceProcessor)

### Accept Uncertainty
**What:** Some tracks can't be assigned definitively.

**Why:** Evidence may be missing (face occluded, timing ambiguous, multiple candidates).

**Approach:**
- Don't force wrong merge to avoid uncertainty
- Mark track with `identity_candidates` set
- Uncertainty persists until evidence resolves it
- System operates with partial timeline
- User can manually resolve if needed

**Example:**
```
# Track could be Agent A or Agent B
track.identity_candidates = {Agent_A, Agent_B}
track.global_agent_id = None  # Uncertain

# Wait for evidence...
# Face match arrives → resolves to Agent_A
# OR process of elimination rules out Agent_B
# OR human reviews and decides
```

---

### Types of Uncertainty

**Identity Uncertainty:**
- Track might belong to Agent A or Agent B
- Example: Two people walk through portal together, tracks split on other side
- Resolution: Face match, process of elimination, human review

**Location Uncertainty:**
- Agent exists but current location unknown
- Example: Person enters building, not visible for 10 minutes
- Resolution: Mark as `at_portal = "garage"` context flag, accept gap in timeline

**Assignment Uncertainty:**
- Can't determine if track belongs to existing agent or is new agent
- Example: Face occluded, timing fits multiple candidates
- Resolution: Queue for LLM/human review

**Vehicle Association Uncertainty:** *(NEEDS MORE SPEC)*
- Person enters vehicle, which person exits?
- Example: Two people enter car, car moves, one person exits (driver or passenger?)
- Strategy: Mark with `inside_vehicle` context flag, use sporadic detections for face matching
- Full spec needed in L3_VehicleOccupancy

---

### Multi-Assignment (No Probabilities)

**What:** Track marked as "possibly Agent A or Agent B" (no probability scores).

**Why:** Can't compute accurate probabilities. Count evidence, don't do arithmetic.

**Approach:**
- `identity_candidates` is a **set** (not probabilities)
- Timeline shows all possibilities
- Process of elimination removes impossible candidates
- Evidence collapses to single agent or requires human review

**Example:**
```
# Not this:
track.probabilities = {Agent_A: 0.65, Agent_B: 0.35}  # ❌

# This:
track.identity_candidates = {Agent_A, Agent_B}  # ✓
```

**Why no probabilities:**
- Can't compute accurate weights (0.3×time + 0.5×face = nonsense)
- Can't debug ("why 0.65?")
- Evidence is countable and explicit

---

### Learn from Corrections

**What:** Human/LLM validations improve future decisions.

**Why:** System learns what "same person" looks like over time.

**Approach:**
- **Face library:** Builds from high-confidence merges only
  - Face recognition ≥ 0.75 threshold
  - OR validated human review
  - Prevents cascading errors (no uncertain merges pollute library)

- **Grid learning:** Transition patterns from validated cross-camera merges
  - Clean data only (single agent, unambiguous)
  - Excludes groups and uncertain cases

- **Face recognition improves:** As library grows, auto-merge rate increases

---

## Strategy for Special Cases

### Concurrent Agents (Same Camera)
**What:** Multiple people visible simultaneously on one camera.

**Why this matters:** Need to keep them separate, track each independently.

**Strategy:**
- YOLO maintains separate tracking IDs (different local_ids)
- TemporalLinker matches each across chunks independently
- IdentityResolver assigns each to different global_agent_id
- No special handling needed (YOLO already separated them)

**Grid learning exclusion:**
- Exclude groups from grid learning (ambiguous which person traveled)
- Only use solo tracks (max_concurrent_agents = 1)
- Keeps grid data clean

---

### Vehicle Occupancy *(NEEDS MORE SPEC)*
**What:** People inside vehicles (hidden or sporadically visible).

**Observable evidence:**
- Person track ends near vehicle (temporal/spatial correlation)
- Vehicle track starts moving
- Sporadic person detections inside vehicle (YOLO through windows)
- Vehicle parks, person track starts nearby (exit correlation)

**Strategy:**
- Mark person track with `inside_vehicle` context flag
- Sporadic detections used for face matching only (not grid learning)
- Accept gaps in person timeline (vehicle track becomes proxy)
- Process of elimination when vehicle parks (who exited?)

**Needs full spec:** See L3_VehicleOccupancy for detailed strategy (TBD).

---

### Structure Entry/Exit *(NEEDS MORE SPEC)*
**What:** People entering buildings/garages (no interior cameras).

**Observable evidence:**
- Track ends at configured portal
- Portal marks boundary (e.g., "garage_door", "front_door")

**Strategy:**
- Mark track with `at_portal` context flag
- Accept location uncertainty (agent in structure)
- When track starts at same portal → could be anyone who entered
- Use face match or process of elimination to resolve

**Needs full spec:** See L3_StructureEntry for detailed strategy (TBD).

---

### Property Boundaries *(NEEDS MORE SPEC)*
**What:** Vehicles/people arriving from outside property (unknown initial state).

**Observable evidence:**
- Track crosses portal marked as `is_property_exit = True`
- Entry from street → unknown initial occupancy

**Strategy:**
- Mark with `property_entry` context flag
- Vehicle arriving from street → occupancy uncertain (could have hidden people)
- Accept uncertainty until people exit vehicle on property

**Needs full spec:** Configuration of property exit portals, how to mark them.

---

## Timeline Construction

**Goal:** "What happened today?" answered by querying LocalTracks, not camera-by-camera logs.

**Approach:**
```python
# Get agent timeline
timeline = db.query(LocalTrack).filter(
    global_agent_id == Agent_A
).order_by(start_time).all()

# Display as journey
for track in timeline:
    print(f"{track.camera_id}: {track.start_time} - {track.end_time}")
    if track.at_portal:
        print(f"  → Entered {track.at_portal}")
    if track.inside_vehicle:
        print(f"  → In vehicle (sporadic)")
```

**Key insight:** Timeline is just ordered LocalTracks. No separate timeline object.

---

## What Gets Filtered vs What Gets Tracked

### Filtered Out (Never Creates LocalTrack):
- Static frames (motion gate)
- Brief detections (< 3s duration)
- Low confidence objects (< threshold)
- Stationary objects (parked car, tree)
- Detector glitches

### Tracked But Excluded from Grid Learning:
- Groups (multiple people concurrent)
- Ambiguous merges (uncertain identity)
- Sporadic vehicle detections (person in car)
- Low-quality tracks (partial visibility)

### Tracked AND Used for Grid Learning:
- Solo tracks (max_concurrent_agents = 1)
- High-confidence merges (face match ≥ 0.75 or validated)
- Sustained movement (≥ 3s duration)
- Clean observations (no occlusions, good lighting)

**Why distinction matters:** Grid learns from clean data only. Prevents noise pollution.

---

## Principles (from L2_Strategy_Principles)

All strategy decisions follow these core principles:

1. **Agent-Centric Thinking** - Tracks belong to agents (people), not cameras
2. **Observable Evidence Only** - No statistical priors, no routine modeling
3. **Learn from Data, Then Refine** - Grid learns automatically, config adds semantics
4. **Count Evidence, Don't Do Arithmetic** - No probability scoring
5. **Human-in-Loop Initially** - Trust is earned, automation via learning
6. **Acceptable Errors** - Not all mistakes matter equally
7. **Learn from Clean Data** - Only high-quality observations train the system

See [L2_Strategy_Principles.md](L2_Strategy_Principles.md) for details.

---

## System Lifecycle

The system's lifecycle is defined by:
1. **Bootstrapping** (see L5_Bootstrap.md) - Initial setup, first data collection
2. **Phased Deployment** (see L6_Deployment.md) - Progressive feature rollout

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - Technical architecture (tracks + handlers)
- **L2_Strategy_Principles.md** - Core principles guiding all decisions

### Mission & Complexity
- **L1 (Mission):** Why we're solving this problem
- **L4 (Complexity):** Where this strategy gets complicated

### Tactics & Implementation
- **L3_ChunkProcessing:** How ChunkProcessor creates tracks (TBD)
- **L3_TemporalLinking:** How TemporalLinker connects tracks (TBD)
- **L3_IdentityResolution:** How IdentityResolver assigns agents (TBD)
- **L3_EvidenceProcessing:** How EvidenceProcessor resolves uncertainty (TBD)
- **L3_VehicleOccupancy:** Vehicle association strategy (NEEDS MORE SPEC)
- **L3_StructureEntry:** Structure entry/exit strategy (NEEDS MORE SPEC)

### Technical Specs
- **L10 (Nomenclature):** Terminology definitions
- **L12_Merging:** Cross-camera merge algorithm
- **L12_ProcessOfElimination:** Elimination logic details
- **L13_Detection:** LocalTrack data structure and creation
