# Level 12 - Process of Elimination
## A System-Wide Pattern for Resolving Ambiguity

## Purpose

**Process of Elimination** is a fundamental reasoning pattern used throughout the MarengoCam system to resolve identity and assignment uncertainty without requiring definitive evidence. Rather than forcing a decision, the system gathers all plausible candidates and systematically rules out impossible options. When only one candidate remains, that is the answer.

This document explains:
- **What** process of elimination is
- **Where** it applies in the system
- **How** each component implements it
- **Why** this approach is superior to forced decisions or probabilistic scoring

## 1. Core Concept

### Definition

**Process of Elimination:** Given an unknown assignment (e.g., "Which agent does this track belong to?"), identify all plausible candidates, then apply constraints to rule out impossible options. The remaining candidate(s) are the answer.

### Key Principle: Single-Source Bias

The system operates under a **single-source bias**: absent contradictory evidence, we assume:
- A track belongs to exactly one agent
- A person in a vehicle is exactly one person
- An entry point at a camera has exactly one source camera

This bias is justified because:
- **Reality:** In most cases, agents move independently
- **Noise filtering:** Cascading uncertainty creates false positives
- **Simplicity:** Single-source decisions are auditable and reversible

### Evidence vs. Constraints

- **Evidence:** Positive indicators (face match, clothing similarity, grid timing)
- **Constraints:** Negative indicators (agent seen elsewhere, impossibly fast travel, conflicting observations)

Process of elimination primarily uses **constraints** to rule out candidates. It's the inverse of evidence-gathering.

## 2. Where Process of Elimination Applies

### 2.1 Merge Evaluation (Track Merging Engine)

**Context:** Two tracks from different cameras are candidates for merging. Is track A the same agent as track B?

**Plausible candidates:**
- Track A could be agent X
- Track A could be agent Y
- Track A could be a new agent (Unknown-N)

**Constraints applied:**
1. **Timing**: Time gap between track A exit and track B entry fits learned grid timing
2. **Alibi check**: Could track B have come from another source camera?
   - If no alternative source exists → agent at A exit and B entry must be the same (deterministic)
   - If alternative sources exist → ambiguous, route to LLM
3. **Spatial**: Exit cell and entry cell are plausible endpoints (learned via grid)
4. **Identity bounds**: How many agents could plausibly be at both locations?

**Example:**
```
Track A: Person exits front camera at T=10s, cell (5,2)
Track B: Person enters back camera at T=13s, cell (0,3)

Plausible candidates for Track A:
  - Agent Person-1 (known, seen elsewhere)
  - Agent Person-2 (known, possible location)
  - Agent Unknown-5 (unassigned track)

Alibi check: "Could anyone else have reached back camera entry cell in 3 seconds?"
  - Person-1: Last seen at East gate, would need 8 seconds to reach back entry → ruled out
  - Person-2: Could have taken south path in 3 seconds → possible
  - Unknown-5: Could have originated anywhere → possible

Result:
  - If only Person-2 is possible → auto-merge (process of elimination)
  - If multiple agents possible → REVIEW_LLM (ambiguous)
```

**Implementation:** L12_Merging.md `evaluate_merge_candidate()`, called by Track Merging Engine

---

### 2.2 Agent Promotion (Track State Manager)

**Context:** A candidate track is promoted from Armed → Active state. Which agent should it belong to?

**Plausible candidates:**
- Previous chunk continuation (same camera, temporal continuity)
- Cross-camera merge (temporal proximity + spatial plausibility)
- New agent (Unknown-N)

**Constraints applied:**
1. **Temporal continuity**: If from same camera and time gap < threshold → previous chunk continuation
2. **Cross-camera plausibility**: If from different camera, does grid allow it?
3. **Identity bounds**: How many agents fit all constraints?

**Example:**
```
Candidate track: Person detected on back camera at T=15s

Previous chunk track on back camera ended at T=12s, belonged to Agent-1
↓ (temporal gap = 3s, same camera)
→ Strong evidence for continuation → assign to Agent-1

Alternative: Could be cross-camera merge from:
  - Front camera track ended at T=11s → grid says 5s typical, 3s gap too fast → ruled out
  - Side camera track ended at T=13s → grid says 3s typical, 2s gap plausible → possible

Result:
  - If only Agent-1 fits → assign to Agent-1 (deterministic)
  - If multiple agents fit → mark as multi-assignment {Agent-1, Side-Agent}, needs disambiguation
```

**Implementation:** L13_TrackStateManager.md `promote_candidate_to_agent()`

---

### 2.3 Multi-Assignment Resolution

**Context:** A track has uncertain assignment: `candidate_agents = [Agent-A, Agent-B, Agent-C]`. Can we narrow the list?

**Plausible candidates:** The list provided by merge evaluation or promotion

**Constraints applied:**
1. **Temporal**: Agent seen elsewhere at overlapping time → ruled out
2. **Spatial**: Agent location incompatible with track entry point → ruled out
3. **Identity bounds**: Accumulate constraints across multiple observations
4. **Portal state**: Agent known to be in structure/vehicle → affects plausibility

**Example:**
```
Track: Unknown person detected at back camera, T=15s
Candidate agents: [Person-1, Person-2, Unknown-5]

Constraint check:
- Person-1: Confirmed visible on front camera at T=14s, still there at T=16s
  → Couldn't have traveled to back camera in 1 second → ruled out
- Person-2: Last seen entering garage at T=12s, state=in_structure →
  → Plausible to exit garage and reach back camera in 3s → still possible
- Unknown-5: New agent, no constraints → still possible

Result after constraint:
- Candidates reduced: [Person-2, Unknown-5]
- System accepts uncertainty, timeline shows: "Person-2 (or Unknown-5) detected at back"
```

**Implementation:**
- Inference during track lifecycle (continuous refinement)
- Could live in Track State Manager or a dedicated Disambiguation Engine
- Called opportunistically as new evidence arrives

---

### 2.4 Vehicle Occupancy Resolution

**Context:** Multiple people enter a vehicle. Later, one person exits. Which person is it?

**Plausible candidates:**
- Any person who entered the vehicle and hasn't been confirmed exiting elsewhere

**Constraints applied:**
1. **Temporal**: Person confirmed visible elsewhere after vehicle park time → ruled out
2. **Spatial**: Person's last known location incompatible with vehicle exit point → ruled out
3. **Identity bounds**: How many people could still be in vehicle?
4. **Physical**: Vehicle interior detections (face, color) match any occupant?

**Example:**
```
Vehicle trace:
- T=10s: Person-1 enters vehicle at front (track ends)
- T=11s: Person-2 enters vehicle at rear (track ends)
- T=15s: Vehicle parks at garage
- T=16s: Person exits vehicle at driver door (new track starts)

Occupants plausible in vehicle: [Person-1, Person-2, Unknown-6]

Constraint check:
- Person-1: No sightings elsewhere, still plausible
- Person-2: Confirmed on back camera at T=17s, exited structure
  → Couldn't have been in vehicle AND visible on back camera simultaneously → ruled out
- Unknown-6: Possible but unlikely (no evidence of entry)

Result:
- Person-1 most likely exited vehicle
- System marks: "Person-1 exited vehicle" with note "Person-2 status ambiguous"
```

**Implementation:**
- Part of Timeline Reconstruction component
- Queries vehicle occupancy state (who entered, who's still inside)
- Applies temporal/spatial constraints
- May require LLM adjudication if ambiguous

---

### 2.5 Track Merging Across Portals

**Context:** Track A disappears into a portal (building, garage). Track B later emerges from the same portal. Are they the same agent?

**Special case:** Portals break normal grid-based timing (time inside is unknown)

**Plausible candidates:**
- Same agent (A = B)
- Different agents (A ≠ B, both used same portal)

**Constraints applied:**
1. **Portal identity**: Both tracks use the same portal (exit cell ≈ entry cell)
2. **Temporal window**: Plausible transit time for that portal type
   - Building entrance: 1-30 minutes
   - Garage: 1-10 minutes
   - Dense vegetation: 1-5 minutes
3. **Physical evidence**: Re-ID score, clothing, vehicle association
4. **Identity bounds**: How many agents could plausibly use this portal in this timeframe?

**Example:**
```
Track A: Person enters building at T=100s, state=in_structure(Building-1)
Track B: Person exits same building at T=220s, 2 minutes later

Occupants in Building-1 at T=220s: [Person-A, Unknown-7]

Constraint check:
- Person-A: Entered building, no other exit confirmed → plausible
- Unknown-7: Possible but no entry evidence

Re-ID check: Track A and B have 0.85 similarity → high confidence

Result:
- Person-A most likely (portal + plausible time + high Re-ID)
- System decides: AUTO_MERGE with reason "portal_reentry_with_reid"
```

**Implementation:** L12_Merging.md `evaluate_merge_candidate()` with portal-specific logic

---

## 3. System-Level Workflow

### 3.1 How Constraints Flow Through Components

```
┌──────────────────────────────────────────────────────────┐
│ Event: New track created or track candidate identified  │
└──────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────┐
│ Evidence Engine:                                         │
│ - Gather plausible candidates from all sources         │
│ - Compute grid constraints (timing, spatial)           │
│ - Run alibi checks (alternative sources?)              │
│ - Return: EvidenceReport with candidates + constraints │
└──────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────┐
│ Track Merging Engine:                                    │
│ - Receive EvidenceReport                               │
│ - Apply decision hierarchy:                            │
│   1. High-confidence face match? → AUTO_MERGE          │
│   2. Only one candidate left after constraints? →      │
│      AUTO_MERGE (process of elimination)               │
│   3. Multiple candidates still plausible? →            │
│      REVIEW_LLM                                        │
│   4. No plausible candidates? → REJECT                 │
│ - Return: MergingDecision (AUTO_MERGE / REVIEW / etc.) │
└──────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────┐
│ Track State Manager:                                     │
│ - Execute merging decision                             │
│ - If AUTO_MERGE: link tracks to same agent            │
│ - If REVIEW_LLM: add to review queue, mark            │
│   track.candidate_agents = [...]                       │
│ - If REJECT: create new agent                          │
│ - Continuous: Apply opportunistic constraints to       │
│   narrow down multi-assignment lists as new evidence   │
│   arrives (e.g., person seen elsewhere)                │
└──────────────────────────────────────────────────────────┘
```

### 3.2 When Does Process of Elimination Succeed?

Process of elimination succeeds when exactly **one candidate remains** after all constraints are applied.

**Scenarios:**
1. ✅ **Temporal overlap rules out all but one** → Merge
2. ✅ **Alibi check finds no alternatives** → Merge
3. ✅ **Agent location incompatible with track** → Mark for review
4. ✅ **Multiple constraints combine to eliminate all but one** → Merge

**Scenarios requiring escalation:**
1. ❓ **Multiple candidates satisfy all constraints** → REVIEW_LLM (ambiguous)
2. ❓ **No candidates satisfy constraints** → REJECT (new agent, or constraint error)
3. ❓ **Conflicting evidence** (e.g., grid says merge, face says different) → REVIEW_HUMAN

## 4. Implementation Patterns

### 4.1 Constraint Checking (Generic Pattern)

Every component that applies process of elimination should follow this pattern:

```python
def apply_process_of_elimination(candidates: List[Agent], constraints: ConstraintSet) -> Result:
    """
    Apply constraints to narrow candidate list.
    Returns: (remaining_candidates, ruled_out_reasons)
    """
    remaining = candidates.copy()
    ruled_out = {}

    for constraint in constraints:
        newly_ruled = []
        for candidate in remaining:
            if constraint.violates(candidate):
                newly_ruled.append(candidate)
                ruled_out[candidate] = constraint.reason()

        remaining = [c for c in remaining if c not in newly_ruled]

    return remaining, ruled_out
```

**Example constraint types:**
- `TemporalConstraint`: Agent location + timing
- `SpatialConstraint`: Geographic impossibility
- `IdentityConstraint`: Known conflicts
- `PortalConstraint`: Portal entry/exit matching
- `AlibiConstraint`: Agent seen elsewhere

### 4.2 Uncertainty Representation

When process of elimination narrows but doesn't fully resolve:

```python
class Track:
    agent_id: Optional[str]  # If resolved
    candidate_agents: List[str]  # If unresolved (ordered by confidence)
    elimination_reasons: Dict[str, str]  # Why candidates were ruled out

    @property
    def is_resolved(self) -> bool:
        return len(self.candidate_agents) == 1

    @property
    def is_ambiguous(self) -> bool:
        return len(self.candidate_agents) > 1
```

### 4.3 Learning from Resolution

When process of elimination succeeds (and later confirmed), system learns:

```python
def learn_from_resolved_track(track: Track, confirmed_agent: str):
    """
    Track was ambiguous, now confirmed. Extract patterns.
    """
    # Which constraints matter most?
    # Can we improve grid learning?
    # Did face recognition confirm elimination?
```

## 5. Comparison to Alternatives

### Alternative 1: Probabilistic Scoring
**"Assign each candidate a probability, merge if >0.85"**

**Why we don't use this:**
- Probabilities require calibration (where does 0.85 come from?)
- System becomes black-box (why did it choose agent B over A?)
- Cascading errors: uncertain assignments used to make next decision
- Hard to debug: "it was 0.84 confidence"

**Process of elimination advantage:**
- Auditable: "We ruled out X, Y, Z, only A fits"
- Conservative: Only decide when necessary
- Reversible: If new evidence arrives, reconsider

### Alternative 2: First-Match (No Elimination)
**"Just merge with first plausible candidate"**

**Why we don't use this:**
- Order-dependent (which candidate listed first?)
- Cascading errors: wrong first choice affects all downstream decisions
- No use of available constraints

**Process of elimination advantage:**
- Uses all available evidence
- Same result regardless of candidate order
- Scales to many cameras/agents

## 6. Strategic Importance

### Why This Pattern Matters

1. **Scalability**: As cameras increase (5 → 10 → 20), constraint-based elimination scales better than matching-based approaches

2. **Auditability**: When a strange merge occurs, we can reconstruct which constraints were applied and why

3. **Reversibility**: If process of elimination selects the wrong candidate, new evidence can change the decision

4. **User Control**: Constraints are explicit in configuration (portal definitions, timing windows, etc.), not hidden in model weights

5. **Uncertainty Management**: System can live with ambiguity ("Agent A or B") without forcing wrong decisions

### Integration with Evidence Hierarchy

Process of elimination is **NOT** the only decision mechanism. It works together with:

1. **Face Recognition** (highest confidence)
   - Definitive: "100% same person" → AUTO_MERGE (no elimination needed)

2. **Grid + Alibi Check** (high confidence)
   - Plausible timing + no alternatives → AUTO_MERGE via process of elimination
   - Plausible timing + alternatives exist → REVIEW_LLM

3. **LLM Adjudication** (medium confidence)
   - For ambiguous cases where process of elimination narrows but doesn't resolve
   - LLM chooses between remaining candidates

4. **Human Review** (definitive for training)
   - Resolves conflicting evidence
   - Provides ground truth for learning

## 7. Related Documents

- **L12_TrackAgentRelationship.md**: How tracks map to agents (core model for understanding assignments)
- **L2_Strategy.md**: High-level strategy for using constraints and process of elimination
- **L12_Merging.md**: Implementation of merge evaluation with alibi checking and portal-based merging
- **L12_VehicleOccupancy.md**: Application of process of elimination to vehicle entry/exit disambiguation
- **L3_EvidenceEngine.md**: Fact-gathering for merge candidates
- **L3_TrackMerging.md**: Decision logic for merge evaluation
- **L3_ProcessOfElimination.md**: Non-technical overview (if created)
- **L13_TrackStateManager.md**: Agent promotion and multi-assignment handling with constraint application
