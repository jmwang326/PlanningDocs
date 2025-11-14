# Level 12 - Track-Agent Relationship and Assignment Model
## How Tracks Map to Agents Across Cameras and Time

## Purpose

This document clarifies the fundamental relationship between **Tracks** and **Agents** in the MarengoCam system. The relationship is directional and one-way: **Tracks point to Agents**, not the reverse.

## 1. Core Definitions

### Agent (Persistent Identity)

**Definition:** An agent is a persistent entity that exists across time and multiple cameras. It represents a single, continuous identity (e.g., "The Gardener", "Honda Accord Blue", "Unknown-7").

**Characteristics:**
- One persistent `agent_id` per entity
- Exists across entire session (multiple cameras, hours of time)
- Links to an `identity_id` (face library) if named/recognized
- Has a lifecycle: created → active → archived/merged
- State tracked: location, visible/hidden, in_structure, in_vehicle, etc.

**Database:**
```sql
CREATE TABLE agents (
    agent_id INTEGER PRIMARY KEY,
    agent_class TEXT,              -- 'person', 'vehicle', 'animal'
    created_at TIMESTAMP,
    last_seen TIMESTAMP,
    identity_id INTEGER,           -- FK to named identity (if recognized)
    status TEXT,                   -- 'active', 'archived', 'merged'
    merged_into_agent_id INTEGER   -- If merged into another agent
);
```

### Track (Observable Window)

**Definition:** A track is a single, continuous observation of an object on **one camera**. It has a start time and end time on that specific camera.

**Characteristics:**
- One `track_id` per observable window per camera
- Single camera only (can't be on multiple cameras)
- Temporal bounds: start_time, end_time
- Grid location: entry_cell, exit_cell
- Points to exactly one `agent_id` (or multiple candidates if ambiguous)
- Quality metadata: solo_track, max_concurrent_agents

**Database:**
```sql
CREATE TABLE tracks (
    track_id INTEGER PRIMARY KEY,
    agent_id INTEGER,              -- FK to agents (resolved assignment)
    camera TEXT,                   -- Single camera only
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    entry_cell INTEGER,            -- 6×6 grid position at start
    exit_cell INTEGER,             -- 6×6 grid position at end
    candidate_agents TEXT,         -- JSON "[7,8,9]" if ambiguous
    status TEXT
);
```

### Example

```
Agent: Person-1 ("The Gardener")
  Created: T=0
  State: visible
  Tracks:
    - Track-A: Front Door camera, T=0-5s, entry_cell=5, exit_cell=2
    - Track-B: Side Path camera, T=6-12s, entry_cell=4, exit_cell=3
    - Track-C: Back Patio camera, T=14-25s, entry_cell=1, exit_cell=5
```

Person-1 is ONE agent with THREE tracks (one per camera).

## 2. Assignment Semantics: Track → Agent

### Direction of Assignment

**Tracks point to Agents, not the reverse:**

```python
class Track:
    track_id: str
    camera_id: str
    agent_id: str  # "Points to" the agent
    candidate_agents: List[str]  # If ambiguous, multiple candidates

class Agent:
    agent_id: str
    # Agent does NOT have tracks field
    # Agent is found by querying: SELECT * FROM tracks WHERE agent_id = X
```

**Why this matters:**
- A track always knows which agent it belongs to (or candidates if unsure)
- An agent is found by querying the tracks table
- This prevents bi-directional pointers and keeps relationships clean

### States of Assignment

A track can be in one of three assignment states:

#### State 1: RESOLVED (Single Agent)

Track knows exactly which agent it belongs to.

```python
track.agent_id = "Person-1"
track.candidate_agents = []
```

**When resolved:**
- High-confidence face match
- Alibi check eliminated all but one candidate
- Process of elimination narrowed list to one
- User/LLM explicitly chose one candidate

**Consistency:** `agent_id` is populated, `candidate_agents` is empty

#### State 2: AMBIGUOUS (Multiple Candidates)

Track cannot be assigned to a single agent yet. Multiple agents are plausible.

```python
track.agent_id = None
track.candidate_agents = ["Person-A", "Person-B", "Unknown-7"]
```

**When ambiguous:**
- Multiple candidates passed alibi check
- Evidence insufficient to choose one
- Waiting for additional evidence to resolve

**Consistency:** `agent_id` is NULL, `candidate_agents` is non-empty

#### State 3: UNASSIGNED (No Merge Attempted Yet)

Track has not been evaluated for merging yet. Fresh track, awaiting assignment.

```python
track.agent_id = None
track.candidate_agents = []
```

**When unassigned:**
- Brand new track, just created
- No merge evaluation has run yet
- System hasn't attempted to assign it

**Consistency:** Both `agent_id` and `candidate_agents` are empty

## 3. Lifecycle: How Tracks Get Assigned to Agents

### Phase 1: Track Creation (Candidate → Formal Track)

When a track is promoted from Armed → Active state:

```python
def promote_candidate_to_track(candidate: CandidateTrack) -> Track:
    """
    Promote from ephemeral candidate to formal track.
    Determine initial agent assignment.
    """
    # Check 1: Same camera continuation?
    prev_track = find_previous_chunk_link(candidate)
    if prev_track and prev_track.agent_id:
        # Same agent as previous chunk
        return create_track(agent_id=prev_track.agent_id)

    # Check 2: Should this be a new agent?
    # (Don't assign to existing agents yet - waiting for merge evaluation)
    new_agent = create_agent(agent_type=candidate.agent_type)
    return create_track(agent_id=new_agent.agent_id)
```

**Result:** Track starts with `agent_id` pointing to either:
- Same agent as previous chunk (same camera)
- A brand new, temporary agent (will possibly merge later)

### Phase 2: Merge Evaluation (Cross-Camera Candidate Finding)

The Evidence Engine looks for cross-camera merge candidates:

```python
def find_merge_candidates(track: Track) -> List[Agent]:
    """
    For this track, which existing agents could it belong to?
    Returns list of plausible agents from OTHER cameras.
    """
    # Time window check
    recent_agents = get_agents_seen_recently(time_window=60s)

    # Grid plausibility check
    plausible = [
        agent for agent in recent_agents
        if grid_allows_transition(agent.last_track, track)
    ]

    # Alibi check (alternative sources)
    candidates_after_alibi = [
        agent for agent in plausible
        if not has_plausible_alternative_source(track)
    ]

    return candidates_after_alibi
```

**Result:** List of agents that could plausibly be this track.

### Phase 3: Merge Decision (Track Merging Engine)

The Merging Engine decides whether to merge:

```python
def evaluate_merge(track: Track, candidate_agent: Agent):
    """
    Should track be merged into candidate_agent?
    Updates track assignment if yes.
    """
    evidence = gather_evidence(track, candidate_agent)

    decision = apply_evidence_hierarchy(evidence)

    if decision == AUTO_MERGE:
        # Merge: track now belongs to candidate_agent
        track.agent_id = candidate_agent.agent_id
        track.candidate_agents = []
        return MERGED

    elif decision == REVIEW_LLM:
        # Ambiguous: mark candidates, wait for LLM/user
        track.agent_id = None
        track.candidate_agents = [candidate_agent.agent_id]
        return AMBIGUOUS

    elif decision == REJECT:
        # Don't merge: keep track's original (temporary) agent
        # track.agent_id stays as-is (temporary unknown agent)
        return REJECTED
```

**Result:** Track's assignment is updated:
- MERGED: `agent_id` = candidate, `candidate_agents` = []
- AMBIGUOUS: `agent_id` = NULL, `candidate_agents` = [list]
- REJECTED: `agent_id` stays as temporary agent

### Phase 4: Constraint-Based Narrowing (Opportunistic)

If track is ambiguous, constraints are applied opportunistically:

```python
def apply_elimination_constraints(track: Track):
    """
    Try to narrow down candidate_agents using temporal/spatial/grid constraints.
    """
    remaining = track.candidate_agents.copy()

    for agent_id in track.candidate_agents:
        agent = get_agent(agent_id)

        # TEMPORAL: Agent seen elsewhere at overlapping time?
        if agent.last_seen_time > track.start_time:
            if agent.last_seen_camera != track.camera_id:
                # Temporal conflict - agent can't be two places
                remaining.remove(agent_id)
                continue

        # SPATIAL: Agent in building, can't exit yet?
        if agent.state == "in_structure":
            if track.start_time < agent.earliest_exit_time:
                remaining.remove(agent_id)
                continue

        # GRID: Travel time doesn't fit learned paths?
        if agent.last_track and not grid_allows(agent.last_track, track):
            remaining.remove(agent_id)
            continue

    # Update track with narrowed candidates
    track.candidate_agents = remaining

    if len(remaining) == 1:
        # Resolved!
        track.agent_id = remaining[0]
        track.candidate_agents = []

    elif len(remaining) == 0:
        # All candidates eliminated - keep temporary agent
        pass
```

**Result:** `candidate_agents` list is narrowed. If only one remains, track is RESOLVED.

## 4. Consistency Rules

### Rule 1: Agent Existence

**Invariant:** If `track.agent_id` is set, the agent must exist.

```python
# Valid:
track.agent_id = "Person-1"
agent = db.query("SELECT * FROM agents WHERE agent_id = ?", "Person-1")
assert agent is not None

# Invalid:
track.agent_id = "Person-1"
# But Person-1 doesn't exist in agents table → CONSTRAINT VIOLATION
```

### Rule 2: Mutual Exclusivity

**Invariant:** `agent_id` and `candidate_agents` are mutually exclusive.

```python
# Valid (RESOLVED):
track.agent_id = "Person-1"
track.candidate_agents = []

# Valid (AMBIGUOUS):
track.agent_id = None
track.candidate_agents = ["Person-A", "Person-B"]

# Valid (UNASSIGNED):
track.agent_id = None
track.candidate_agents = []

# INVALID:
track.agent_id = "Person-1"
track.candidate_agents = ["Person-A", "Person-B"]  # Can't have both!
```

### Rule 3: No Orphan Tracks

**Invariant:** Every track must have a temporary agent if no candidates exist.

```python
# A newly created track must have SOME agent_id
# even if it's a temporary "Unknown-7" that will merge later

new_track = Track(
    track_id="Track-42",
    camera="Front",
    agent_id=create_temporary_agent().agent_id,  # Never NULL
    candidate_agents=[]
)
```

**Why:** Prevents orphan tracks with NULL agent_id. Timeline reconstruction requires every track to have an agent.

### Rule 4: Single Camera Per Track

**Invariant:** Each track belongs to exactly one camera.

```python
# Valid:
track.camera = "Front Door"

# Invalid:
track.camera = ["Front Door", "Side Path"]  # A track can't span cameras
```

## 5. Example Workflow: Multi-Camera Merge

### Scenario: Person walks from front camera to side camera

```
T=0s: Person detected on Front Door camera
     → System creates Track-A
     → Creates temporary Agent-99 to hold it
     Track-A: agent_id=Agent-99, candidate_agents=[]

T=5s: Track-A ends (person exits front camera)
     → Agent-99.last_track = Track-A
     → Agent-99.state = "visible"

T=7s: Person detected on Side Path camera
     → System creates Track-B
     → Creates temporary Agent-100 to hold it
     Track-B: agent_id=Agent-100, candidate_agents=[]

T=7-10s: Merge evaluation runs
     Evidence Engine:
     - Finds Track-A as potential source for Track-B
     - Grid says 2 seconds plausible (track gap = 7-5 = 2s)
     - Alibi check: no other camera's track fits timing
     → Returns candidates = [Agent-99]

     Track Merging Engine:
     - Sees only one candidate (Agent-99)
     - No face match needed (single source)
     - Process of elimination: Agent-99 is the only source
     → Decision: AUTO_MERGE

T=10s: Merge applied
     Track-B.agent_id = Agent-99  ← Track points to Agent-99
     Track-B.candidate_agents = []
     Agent-100 is now orphaned (no tracks point to it)

Final state:
     Agent-99 (The actual person):
       - Track-A (Front, T=0-5s)
       - Track-B (Side, T=7-10s)

     Agent-100 (Temporary, no tracks):
       - Can be garbage collected
```

### What if there were multiple candidates?

```
T=7-10s: Merge evaluation runs
     Evidence Engine:
     - Track-A could be source (Agent-99)
     - But also Track-C from East camera also fits timing (Agent-88)
     → Returns candidates = [Agent-99, Agent-88]

     Track Merging Engine:
     - Multiple candidates, ambiguous
     → Decision: REVIEW_LLM

T=10s: Track assigned as ambiguous
     Track-B.agent_id = None  ← Unresolved
     Track-B.candidate_agents = [Agent-99, Agent-88]
     Agent-100 is orphaned

     System waits for:
     - LLM adjudication (look at face/clothing)
     - User resolution (manual choice)
     - Future constraints (Agent-99 seen elsewhere → rules out)

T=15s: Later, Agent-88 is tracked elsewhere (temporal conflict)
     Constraint engine re-runs:
     Track-B.candidate_agents = [Agent-88]  ← Agent-88 eliminated
     → Only Agent-99 remains → RESOLVED
     Track-B.agent_id = Agent-99
     Track-B.candidate_agents = []
```

## 6. Special Case: Temporary Agents

### Why Create Temporary Agents?

Each track needs an `agent_id` immediately (no NULL agents). But we don't know if it's the "right" agent until merge evaluation runs.

**Solution:** Create a temporary agent to hold the track until we know better.

```python
def promote_candidate_to_track(candidate):
    # Don't assign to existing agents yet
    # Create a temporary agent as a placeholder

    temp_agent = Agent(
        agent_id=generate_unknown_id(),  # "Unknown-42"
        agent_type=candidate.agent_type,
        state="active"
    )

    track = Track(
        track_id=candidate.track_id,
        agent_id=temp_agent.agent_id,  # Temporary
        camera=candidate.camera
    )

    return track, temp_agent
```

### When Temp Agents Are Kept

A temp agent becomes permanent if:
1. **No merge candidates found** → It's a new person
2. **All merge candidates eliminated** → It's a new person
3. **User explicitly creates new identity** → Named in face library

```
Track-A: agent_id=Agent-99 (temp)
    → No merge candidates? Keep Agent-99 as "Unknown-7"
    → User identifies as "John Doe"? Rename Agent-99 in face library
```

### When Temp Agents Are Discarded

A temp agent is garbage-collected if:
1. **Track merged into an existing agent** → Reassign track, drop temp agent

```
Track-B: agent_id=Agent-100 (temp)
    → Merge evaluation: should be Agent-99
    → Change: Track-B.agent_id = Agent-99
    → Agent-100 has no tracks → Can be discarded
```

## 7. Related Documents

- **L13_Database.md:** SQL schema for agents, tracks, and relationships
- **L13_TrackStateManager.md:** Lifecycle of tracks and agent promotion
- **L12_Merging.md:** Merge evaluation and candidate finding
- **L12_ProcessOfElimination.md:** Constraint-based narrowing of candidates
