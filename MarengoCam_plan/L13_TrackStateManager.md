# Level 13 - Track State Manager

## Purpose
Technical specifications for managing candidate tracks during Armed state, promoting them to formal tracks, and handling agent state transitions.

## Candidate Track

### Data Structure
```python
class CandidateTrack:
    """Ephemeral track accumulating detections during Armed state."""
    candidate_id: str  # Unique per camera (e.g., "cand_FrontDoor_20251114_120030")
    camera_id: str
    agent_type: str  # "person", "vehicle", "animal"

    # Temporal accumulation
    first_detection_time: float  # UTC timestamp
    last_detection_time: float
    detections: List[Detection]  # Frame-by-frame accumulation

    # Quality metrics
    duration_seconds: float  # Elapsed since first detection
    frame_count: int
    avg_confidence: float
    movement_score: float  # Estimate of sustained motion (0.0-1.0)

    def is_promoted(self) -> bool:
        """Check promotion criteria."""
        return (
            self.duration_seconds >= 3.0 and
            self.movement_score > MOVEMENT_THRESHOLD and
            self.frame_count >= MIN_FRAME_COUNT
        )

    def is_expired(self, now: float, arm_timeout: float) -> bool:
        """Check if ARM state has timed out without promotion."""
        return (now - self.first_detection_time) > arm_timeout
```

### Lifecycle
1. **Creation (Standby → Armed):** Motion gate + sparse YOLO detection → create CandidateTrack
2. **Accumulation (Armed):** Each frame: accumulate detection, update duration, compute movement_score
3. **Promotion (→ Active):** If `is_promoted()` → convert to LocalTrack, create/merge Agent
4. **Discard (→ Standby):** If `is_expired()` or motion stops → delete candidate, return to Standby

### Movement Score Calculation
```python
def compute_movement_score(detections: List[Detection]) -> float:
    """
    Estimate sustained motion from bounding box centers.
    Returns 0.0 (static) to 1.0 (highly dynamic).

    Factors:
    - Max displacement between frames (pixels)
    - Consistency of direction (not jittery)
    - Smoothness (no sudden jumps)
    """
    if len(detections) < 2:
        return 0.0

    positions = [d.center for d in detections]
    displacements = [
        euclidean_distance(positions[i], positions[i+1])
        for i in range(len(positions) - 1)
    ]

    # Average displacement (normalized by frame size)
    avg_displacement = mean(displacements) / FRAME_DIAGONAL

    # Consistency: low variance = smooth motion
    motion_variance = variance(displacements)
    consistency_score = 1.0 / (1.0 + motion_variance)

    # Composite: movement + smoothness
    return min(1.0, avg_displacement * consistency_score)
```

## Agent Promotion
```python
def promote_candidate_to_agent(
    candidate: CandidateTrack,
    existing_agents: Dict[str, Agent]
) -> Agent:
    """
    Promote candidate track to formal agent.

    Cases:
    1. Continuation from previous chunk → inherit agent ID
    2. Cross-camera merge candidate → merge to existing agent
    3. New agent → create Agent(Unknown-N)
    """
    # Check for cross-chunk continuation
    prev_chunk_track = find_previous_chunk_link(candidate)
    if prev_chunk_track and prev_chunk_track.agent_id:
        return existing_agents[prev_chunk_track.agent_id]

    # Check for cross-camera merge
    merge_candidates = find_merge_candidates(candidate, existing_agents)
    if len(merge_candidates) == 1:
        return merge_candidates[0]

    # New agent
    new_agent = Agent(
        agent_id=next_unknown_id(),
        agent_type=candidate.agent_type,
        created_at=candidate.first_detection_time
    )
    return new_agent
```

## Track Data Structure (Agent Assignment)

```python
class Track:
    """A single-camera observation of an agent."""
    track_id: str
    camera_id: str
    agent_type: str

    # Temporal bounds
    start_time: float
    end_time: float
    detections: List[Detection]

    # Grid data
    entry_cell: Tuple[int, int]  # 6x6 grid position at start
    exit_cell: Tuple[int, int]   # 6x6 grid position at end

    # Identity assignment
    agent_id: Optional[str]  # Single resolved agent, or None if ambiguous
    candidate_agents: List[str]  # Multiple plausible agents (if agent_id is None)

    # Tracking confidence and evidence
    confidence: float  # Overall confidence in current assignment
    elimination_log: Dict[str, str]  # Agents ruled out + reason

    @property
    def is_resolved(self) -> bool:
        """Track assignment is definitive."""
        return self.agent_id is not None and len(self.candidate_agents) == 0

    @property
    def is_ambiguous(self) -> bool:
        """Track has multiple candidate agents."""
        return self.agent_id is None and len(self.candidate_agents) > 1

    @property
    def is_unassigned(self) -> bool:
        """No agent assignment attempt made yet."""
        return self.agent_id is None and len(self.candidate_agents) == 0
```

## Multi-Assignment Resolution

**Problem:** A track's assignment is ambiguous: `candidate_agents = [Agent-A, Agent-B, Agent-C]`. The system must progressively narrow the list as evidence arrives.

### Constraint-Based Narrowing (Process of Elimination)

The system applies constraints opportunistically (as evidence arrives) to eliminate impossible candidates:

```python
def apply_elimination_constraints(
    track: Track,
    active_agents: Dict[str, Agent],
    master_grid: InterCameraGrid
) -> Track:
    """
    Apply process of elimination to narrow candidate list.
    Returns updated track with eliminated candidates moved to elimination_log.
    """
    remaining_candidates = track.candidate_agents.copy()

    for agent_id in track.candidate_agents:
        agent = active_agents.get(agent_id)
        if not agent:
            # Agent no longer exists
            track.elimination_log[agent_id] = "agent_deleted"
            remaining_candidates.remove(agent_id)
            continue

        # CONSTRAINT 1: Temporal (agent seen elsewhere at overlapping time)
        if agent.last_seen_time > track.start_time and agent.last_seen_camera != track.camera_id:
            # Agent confirmed visible elsewhere after track start
            elapsed = agent.last_seen_time - track.start_time
            if elapsed < SPATIAL_PLAUSIBILITY_WINDOW:
                # Can't be in two places at once
                track.elimination_log[agent_id] = f"temporal_conflict_{elapsed:.1f}s"
                remaining_candidates.remove(agent_id)
                continue

        # CONSTRAINT 2: Spatial (agent location incompatible)
        if agent.current_state == "in_structure" and agent.structure_id:
            # Agent known to be inside building
            min_exit_time = agent.structure_entry_time + TYPICAL_STRUCTURE_DWELL
            if track.start_time < min_exit_time:
                # Agent can't exit structure yet
                track.elimination_log[agent_id] = "still_in_structure"
                remaining_candidates.remove(agent_id)
                continue

        # CONSTRAINT 3: Grid plausibility (timing doesn't fit learned paths)
        if len(agent.tracks) > 0:
            last_track = agent.tracks[-1]
            travel_time_observed = track.start_time - last_track.end_time

            # Check learned grid
            key = (last_track.camera_id, track.camera_id)
            min_time = master_grid.fastest_inter_camera_time.get(key)

            if min_time is not None:
                # We've learned this path
                margin = GRID_VARIANCE_MARGIN  # e.g., 3.0 seconds
                if travel_time_observed < min_time - margin:
                    # Agent couldn't have traveled that fast
                    track.elimination_log[agent_id] = f"too_fast_{travel_time_observed:.1f}s"
                    remaining_candidates.remove(agent_id)
                    continue

    # Update track
    track.candidate_agents = remaining_candidates

    # Resolve if only one candidate remains
    if len(remaining_candidates) == 1:
        track.agent_id = remaining_candidates[0]
        track.candidate_agents = []
        return RESOLVED
    elif len(remaining_candidates) == 0:
        # All candidates eliminated - new agent
        track.agent_id = create_new_unknown_agent().agent_id
        track.candidate_agents = []
        return RESOLVED_NEW
    else:
        # Still ambiguous
        return AMBIGUOUS

```

### Opportunistic Application

The constraint check runs in two contexts:

1. **Immediately after merge evaluation** (Track Merging Engine completes)
   - Evidence Engine identified merge candidates
   - Before promotion, apply elimination

2. **Continuously during track lifecycle** (Track State Manager)
   - Each time new evidence arrives (agent state changes, track completion)
   - Re-check all ambiguous tracks' candidate lists
   - May resolve tracks that were previously ambiguous

```python
def on_agent_state_changed(agent_id: str, old_state: str, new_state: str):
    """
    Trigger: Agent changes state (e.g., enters building, exits vehicle).
    Action: Re-evaluate all ambiguous tracks that might be affected.
    """
    ambiguous_tracks = get_all_ambiguous_tracks()

    for track in ambiguous_tracks:
        if agent_id in track.candidate_agents:
            # This agent is a candidate for this track
            # Re-apply constraints with new information
            apply_elimination_constraints(track, ...)

            if track.is_resolved:
                # Previously ambiguous track now resolved!
                log_track_resolution(track, agent_id, reason="state_change")
```

### Example Scenario

```
T=10s: Person enters front door, track ends, ambiguous assignment
       candidate_agents = [Person-A (contractor), Person-B (resident), Unknown-7]

T=15s: Person-A tracked on east camera (exit camera)
       → Temporal constraint: Person-A seen 5s after front entry
       → If travel_time(front→east) = 8s, Person-A is too far
       → Eliminate: Person-A ruled out

       Track now: candidate_agents = [Person-B, Unknown-7]

T=20s: Person-B's vehicle (which Person-A was driving) exits west gate
       → Person-A must still be in vehicle (state=in_vehicle)
       → Spatial constraint: Can't be both in vehicle AND on east camera
       → Eliminate: Person-A already eliminated, Person-B now in_vehicle
       → If Person-B is in_vehicle at T=20, can't have been front entry
       → Eliminate: Person-B ruled out

       Track now: candidate_agents = [Unknown-7]
       → RESOLVED: Track belongs to Unknown-7

Timeline result:
  - T=10s: Unknown-7 entered front door
  - T=12-20s: Unknown-7 inside structure (inferred)
  - T=20+: Unknown-7 location unknown (not tracking)
```

## Related Documents
- **L12_ProcessOfElimination:** System-wide pattern for constraint-based disambiguation
- **L12_Merging:** Merge evaluation and cross-camera candidate identification
- **L3_DetectionAndTracking:** Candidate track state machine (non-technical)
- **L3_TrackStateManager:** Non-technical overview of state management
- **L13_Detection:** Chunk processing and LocalTrack creation (uses promoted candidates)
