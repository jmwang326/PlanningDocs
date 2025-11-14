# Level 12 - Concurrent Tracking
## Handling Multiple Agents on the Same Camera

## Status Note

**SUPERSEDED BY L12_YOLOTrackingLeverage.md** - See that document for the preferred approach using YOLO11's native tracking IDs to disambiguate concurrent agents automatically, rather than creating ambiguous GROUP tracks. GROUP track approach (documented here) is a fallback for cases where YOLO tracking is unreliable or when more detailed interaction analysis is needed.

## Purpose

This document specifies how the system handles multiple agents visible simultaneously on a single camera. The challenge: maintaining separate identity threads when agents overlap, interact, and separate multiple times. See L12_YOLOTrackingLeverage for the primary implementation approach.

## 1. Core Scenario

### Example: Gardener + Visitor Interactions

```
T=0s: Gardener enters Front camera alone
      Track-A started (Gardener only)

T=5s: Visitor walks into frame, talks to Gardener
      Both now visible together
      What happens to Track-A?

T=6s: Visitor walks away, out of frame
      Gardener continues alone
      What happens to Track-A?

T=20s: Visitor re-enters frame again, they talk
      Both visible again
      Do we continue Track-A? Start new?

T=21s: Visitor leaves again
      Gardener still there working

T=60s: Gardener finishes and exits
      Track-A ends
```

**Central Question:** How do we represent this as tracks?

## 2. Design Decision: Overlap = New Track

### Principle: Single-Agent Purity

**Rule:** When a second agent enters a view where another is already present, the overlap is treated as a NEW GROUP, not a continuation.

```python
class TrackState:
    SOLO = "solo"          # Single agent, no overlaps
    GROUP_START = "group_start"   # First group moment
    GROUP_ONGOING = "group_ongoing"   # Multiple frames with multiple agents
    GROUP_END = "group_end"   # Last frame of group, then agent leaves
```

### Why This Matters

1. **Grid Learning:** Only `SOLO` tracks are used to train the grid
   - Grouping creates ambiguity: which person traveled from A to B?
   - Excluding groups = clean training signal

2. **Identity Resolution:** Concurrent tracks have uncertain identity
   - Who left the frame? Who stayed?
   - Need process of elimination, not grid predictions

3. **Simplicity:** Matches how humans watch video
   - "I see two people together, then person A leaves"
   - Not: "Person A's track continued through the group"

### Track Segmentation Rule

```python
def should_split_track(track: Track, new_agents_detected: int) -> bool:
    """
    When SOLO track encounters GROUP (multiple agents):
    - If current track was SOLO, mark end and create GROUP track
    - If track is already GROUP, continue (more agents, same complexity)

    Returns: True if track should be ended and new one started
    """
    if track.state == SOLO and new_agents_detected > 1:
        # Transition from SOLO → GROUP
        # Gardener was alone, now visitor appears
        return True

    return False  # Continue track
```

## 3. Track Lifecycle with Concurrency

### Phase 1: Solo Activity

```
T=0-5s: Gardener alone
        Track-A (agent_id=Gardener, state=SOLO)
        - entry_cell = (2,3)
        - Detections at T=0,1,2,3,4,5
        - max_concurrent_agents = 1
        - solo_track = TRUE
```

### Phase 2: Group Entry

```
T=5s: Visitor enters frame (second agent detected)

Decision point: Should we end Track-A?
- Track-A is SOLO (max_concurrent_agents = 1)
- Now detecting 2 agents
- **ACTION: End Track-A, start Track-GROUP**

Track-A ends:
  - end_time = 5s
  - final_cell = (2,3)
  - solo_track = TRUE  ← clean for grid learning

Track-GROUP-1 starts:
  - start_time = 5s
  - participants = [Gardener, Visitor]
  - agent_id = ??? (ambiguous)
  - candidate_agents = [Gardener, Visitor]
  - max_concurrent_agents = 2
  - solo_track = FALSE
```

### Phase 3: Group Continuation

```
T=5-6s: Both agents visible, talking

Track-GROUP-1 continues:
  - Detections accumulate for both agents
  - Could track bounding box of each? Or group as unit?

Options:
A. Track each agent separately within group
   - Sub-track: Gardener-sub within GROUP-1
   - Sub-track: Visitor-sub within GROUP-1
   - More granular but complex

B. Track group as unit
   - Treat as "interaction unit"
   - Record: who was involved, duration, location
   - Simpler, sufficient for timeline
```

### Phase 4: One Agent Leaves (Visitor)

```
T=6s: Visitor walks out of frame

Decision point: What happens to Track-GROUP-1?

Option 1: End GROUP track, resume SOLO track for Gardener
  - Simpler: straight continuation of Gardener's presence
  - Tracks: A (SOLO, 0-5s) → GROUP-1 (5-6s) → B (SOLO, 6-60s)

Option 2: Keep GROUP track ending, note exit
  - Track-GROUP-1 ends with: "Visitor exited, Gardener stayed"
  - Start Track-B for Gardener's solo continuation

Option 3: Unified long track with annotations
  - Track-GARDENER spans T=0-60s
  - Annotated: "solo 0-5s, group 5-6s, solo 6-20s, group 20-21s, solo 21-60s"
  - Requires new data structure

**Recommended: Option 1 (Simple continuation)**
```

```python
def handle_group_exit(track_group: Track, exiting_agent: str, remaining_agents: List[str]):
    """When one agent leaves a group while others stay."""

    if len(remaining_agents) == 1:
        # Group dissolves back to SOLO
        track_group.end_time = current_time
        track_group.exit_cell = get_position(remaining_agents[0])

        # Create new SOLO track for remaining agent
        new_solo_track = Track(
            agent_id=remaining_agents[0],
            start_time=current_time,
            state=SOLO,
            entry_cell=track_group.exit_cell
        )
        return track_group, new_solo_track

    elif len(remaining_agents) > 1:
        # Still a group, just smaller
        track_group.participants = remaining_agents
        return track_group, None
```

### Phase 5: Visitor Returns (Same or Different Visit?)

```
T=20s: Visitor re-enters frame

Question: Is this the SAME visitor or a DIFFERENT person who looks similar?

Database fields to distinguish:
- If same visitor (same Re-ID): re-entry scenario
- If different person: new agent, new identity

Track-B (Gardener solo) ends at T=20s

Track-GROUP-2 starts at T=20s:
  - participants = [Gardener, Visitor]
  - But which Visitor? Agent-2 (re-visit) or Agent-3 (new)?
  - Use Re-ID similarity to determine
```

## 4. Group Track Representation

### Data Structure

```python
class Track:
    # Standard fields
    track_id: str
    camera: str
    start_time: float
    end_time: float

    # Solo vs Group
    state: str  # SOLO, GROUP_START, GROUP_ONGOING, GROUP_END
    solo_track: bool  # For grid learning (TRUE only if always solo)
    max_concurrent_agents: int  # Max agents ever visible together

    # Group-specific
    participants: List[str]  # Agent IDs in group (may be ambiguous)
    group_id: int  # Links tracks from same interaction group

    # If group/ambiguous
    agent_id: Optional[str]  # NULL if group
    candidate_agents: List[str]  # If group, may not be assignable

    # Tracking individual positions within group
    sub_tracks: Optional[List[SubTrack]]  # Optional per-agent tracking
```

### Sub-Track (Optional, for Detailed Analysis)

```python
class SubTrack:
    """Fine-grained tracking of one agent within a group."""
    agent_id: str
    group_track_id: str
    time_window: Tuple[float, float]  # When this agent was in the group
    bounding_boxes: List[BBox]  # Agent's position within frames
    entry_cell: int  # Where in frame did they appear
    exit_cell: int  # Where did they leave from
```

**When to use SubTracks:**
- ✅ Multi-agent interaction analysis (who was where in group)
- ✅ Detailed timeline reconstruction
- ❌ Grid learning (excluded because of ambiguity)
- ❌ Basic merge decisions (use solo tracks only)

## 5. Decision Logic for Overlapping Tracks

### When to Create Group Track

```python
def detect_group_start(frame_detections: List[Detection], active_tracks: List[Track]) -> bool:
    """
    Detect transition from SOLO to GROUP.
    """
    unique_agents = detect_unique_agents(frame_detections)
    solo_tracks = [t for t in active_tracks if t.state == SOLO]

    if len(unique_agents) > len(solo_tracks):
        # More agents detected than currently tracked
        # Some agents are new entrants (group forming)
        return True

    if len(unique_agents) > 1 and len(solo_tracks) == 1:
        # Was tracking 1 agent solo, now detecting 2+
        # SOLO track must end, GROUP must begin
        return True

    return False
```

### When to End Group Track

```python
def detect_group_end(frame_detections: List[Detection], group_track: Track) -> bool:
    """
    Detect when group dissolves back to solo or breaks up.
    """
    unique_agents = detect_unique_agents(frame_detections)

    # Group ends if we're back to single agent
    if len(unique_agents) == 1:
        return True

    # Group ends if all tracked agents left
    if len(unique_agents) == 0:
        return True

    return False
```

## 6. Merging and Grid Learning

### Grid Learning: SOLO Tracks Only

```python
def can_use_for_grid_learning(track: Track) -> bool:
    """
    Only pristine SOLO tracks used for grid training.
    Excludes group tracks entirely.
    """
    if not track.solo_track:
        return False
    if track.max_concurrent_agents > 1:
        return False
    if track.candidate_agents and len(track.candidate_agents) > 0:
        return False

    return True  # Clean signal
```

**Implication:** Group tracks never contribute to grid learning.

### Merging: Group Tracks Are Ambiguous

```python
def evaluate_merge_for_group_track(group_track: Track, candidate_agent: Agent):
    """
    Group track = multiple agents, ambiguous identity.
    Cannot merge definitively.
    """
    if not group_track.state.startswith("GROUP"):
        # Solo track, use normal merge logic
        return evaluate_merge_candidate(group_track, candidate_agent)

    # Group track: multiple possible agents
    # Cannot merge to single agent_id
    return AMBIGUOUS, "group_track_identity_unclear"
```

## 7. Timeline Reconstruction with Groups

### How to Display Groups

```python
def timeline_event_for_group(group_track: Track):
    """Convert group track to human-readable event."""
    participants = group_track.participants
    duration = group_track.end_time - group_track.start_time

    if len(participants) == 2:
        agent_a, agent_b = participants
        return f"{agent_a.name} and {agent_b.name} interacted for {duration:.0f}s"

    else:
        names = ", ".join([a.name for a in participants])
        return f"Group activity: {names} (duration {duration:.0f}s)"
```

### Gardener + Visitor Timeline Example

```
T=0-5s:   "Gardener entered Front Door, began working"
T=5-6s:   "Visitor arrived and spoke with Gardener"
T=6-20s:  "Gardener continued working alone"
T=20-21s: "Visitor returned and spoke with Gardener"
T=21-60s: "Gardener continued working alone"
T=60s:    "Gardener left Front Door"
```

## 8. Identity Resolution for Groups

### The Ambiguity Problem

When Visitor appears and disappears multiple times:
- T=5-6s: Is this "Visitor-A"?
- T=20-21s: Is this same "Visitor-A" returning, or "Visitor-B" (different person)?

### Solution: Re-ID Confidence Threshold

```python
def resolve_group_participant_identity(frame_detections: List[Detection], group_track: Track):
    """
    Use Re-ID to determine if a returning agent is the same person.
    """
    if group_track.state != GROUP_START:
        return  # Only resolve at start

    new_detections = [d for d in frame_detections if d not in group_track.detections]

    for new_agent_det in new_detections:
        face_sim = compute_face_similarity(new_agent_det.face, known_faces)
        reid_sim = compute_reid_similarity(new_agent_det.reid, known_embeddings)

        if face_sim > 0.8 or reid_sim > 0.8:
            # High confidence: same person as before
            # Link to same agent_id
            match_agent = find_agent_by_similarity(face_sim, reid_sim)
            group_track.participants[new_agent_det] = match_agent
        else:
            # New person
            new_agent = create_unknown_agent()
            group_track.participants[new_agent_det] = new_agent
```

## 9. Implementation Considerations

### Challenge 1: YOLO Tracking IDs

**Problem:** YOLO provides tracking IDs within chunks, but groups complicate this.

```
Frame 1: YOLO_ID=1 (Gardener), YOLO_ID=2 (Visitor)
Frame 2: YOLO_ID=1 (Gardener), YOLO_ID=2 (Visitor)  ← same IDs
Frame 3: YOLO_ID=1 (Gardener)  ← Visitor leaves
Frame 4: YOLO_ID=1 (Gardener)  ← still same
Frame 5: YOLO_ID=1 (Gardener), YOLO_ID=3 (New Agent)  ← different ID for Visitor?
```

**Solution:** Use spatial consistency (proximity, bounding box overlap) + Re-ID to match agents across re-entries, not just YOLO IDs.

### Challenge 2: Sub-Track Tracking

**Problem:** Tracking exact positions of multiple agents within a group frame is complex.

**Solution:** Make sub-tracks optional:
- Required for: Advanced analysis, interaction patterns
- Optional for: Basic timeline reconstruction
- Never required for: Grid learning or merge decisions

```python
# Simplified group track (sufficient)
group_track = Track(
    participants=[Gardener, Visitor],
    group_id=1,
    state="GROUP_ONGOING"
)

# Detailed tracking (optional, for rich analysis)
sub_tracks = [
    SubTrack(agent=Gardener, bboxes=[...]),
    SubTrack(agent=Visitor, bboxes=[...])
]
```

### Challenge 3: Concurrent Frames

**Problem:** YOLO might detect both agents in same frame, or detector noise.

**Solution:** Require persistence:
- Group only forms if multiple agents visible in 3+ consecutive frames
- Single-frame overlap = noise, ignore
- Threshold prevents chatter from detector jitter

```python
def form_group_track(concurrent_detection_frames: int):
    """
    Only form group if multiple agents visible for threshold duration.
    """
    PERSISTENCE_FRAMES = 3
    if concurrent_detection_frames < PERSISTENCE_FRAMES:
        return None  # Ignore noise

    # Confirmed group activity
    return Group(...)
```

## 10. Related Documents

- **L13_Database.md:** Schema fields for `solo_track`, `max_concurrent_agents`, `group_id`
- **L12_Merging.md:** How group tracks affect merge decisions (ambiguous, excluded from grid learning)
- **L12_TrackAgentRelationship.md:** Assignment model (groups have `candidate_agents`, no `agent_id`)
- **L2_Strategy.md:** Grid learning only from validated merges (exclude groups for cleanliness)
- **L13_TimelineReconstruction.md:** How to convert group tracks to human-readable events
