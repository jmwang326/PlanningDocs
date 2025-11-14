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

## Related Documents
- **L3_DetectionAndTracking:** Candidate track state machine (non-technical)
- **L13_Detection:** Chunk processing and LocalTrack creation (uses promoted candidates)
