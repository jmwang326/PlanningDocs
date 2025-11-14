# Level 12 - Leveraging YOLO11 Tracking for Concurrent Agent Disambiguation
## Using YOLO Local IDs to Disambiguate Overlapping Agents

## Purpose

This document specifies how to **use YOLO11's native tracking IDs to keep overlapping agents separated**, rather than creating ambiguous GROUP tracks. The key insight: YOLO11 already solves the hard problem (maintaining consistent object identity frame-to-frame), so we should preserve that information and use it to build unambiguous individual timelines.

## 1. The Problem with Pure Merging Approach

### Current Design (Previous)
```
When Gardener (agent already tracked) + Visitor (new person) appear together:
- Create GROUP track (ambiguous identity)
- Can't use for grid learning (multiple people, unclear which went where)
- Requires LLM/manual review to separate

Result: Clear individual timelines ONLY after user resolves ambiguity
```

### Better Approach: Preserve YOLO Identity
```
YOLO11 maintains tracking ID within each 12s chunk:
- YOLO_ID=1: Gardener (appears frame 0-500)
- YOLO_ID=2: Visitor (appears frame 150-200)

Within chunk: We KNOW they're separate entities
Question: Can we leverage this separation to avoid ambiguity?

Goal: Build separate tracks for Gardener and Visitor automatically,
      NO ambiguity needed, NO group track
```

## 2. Additional Information Needed

### What We Have (Current L13_Detection)
```python
class LocalTrack:
    local_id: int          # YOLO tracker ID (within chunk)
    detections: List[Detection]  # Frame-by-frame data
    global_agent_id: Optional[int]  # Mapped to global agent (if known)
```

### What We Need to Add

#### 2.1 YOLO Tracking Confidence

```python
class LocalTrack:
    # NEW FIELDS:

    # YOLO tracking quality
    tracking_confidence: float  # How "confident" is YOLO this is same object?
                                 # (Higher = more stable tracking)

    tracking_interruptions: int  # How many frames was ID lost/reassigned?
                                  # (Lower = more reliable)

    id_stability_score: float   # Custom metric: consistency of ID assignment
                                 # = (frames_with_id / total_frames)
                                 # = 1.0 if uninterrupted, <1.0 if lost/regained

    # Distinguish track origins
    is_yolo_tracked: bool      # TRUE if YOLO continuously tracked (best case)
    is_reacquired_after_loss: bool  # TRUE if ID was lost then regained
                                     # (less reliable, may be different person)
```

**Why these fields?**
- YOLO11 tracking is good but not perfect
- If YOLO loses a person for several frames, the re-acquired ID might be wrong
- We need to track confidence in the YOLO tracking itself, not just detection

#### 2.2 Visual Similarity Within Concurrent Period

```python
class LocalTrack:
    # NEW FIELDS for concurrent agents:

    # When track overlaps with other tracks on same camera
    concurrent_tracks: List[int]  # YOLO local IDs of agents visible simultaneously

    visual_similarity_to_others: Dict[int, float]
    # E.g.: {
    #   1: 0.95,  # High similarity to YOLO_ID=1 (probably same person reappearing)
    #   2: 0.12,  # Low similarity to YOLO_ID=2 (definitely different person)
    # }

    # Clothing consistency across concurrent period
    dominant_color: Tuple[int, int, int]  # RGB color of dominant clothing
    color_histogram: bytes  # Full histogram (for detailed comparison)
    pose_signature: bytes  # Body pose/silhouette (for gait analysis, future)
```

**Why these fields?**
- When YOLO says "YOLO_ID=1 and YOLO_ID=2", we can verify they're truly different
- Visual similarity helps confirm YOLO's tracking (high confidence case)
- Can detect if YOLO made a tracking error (assigns same person two different IDs)

#### 2.3 Spatial Separation Metrics

```python
class LocalTrack:
    # NEW FIELDS: How separate are concurrent agents?

    min_distance_to_others: Dict[int, float]
    # E.g.: {
    #   1: 0.45,  # Minimum pixel distance to YOLO_ID=1 (45% of frame width)
    #   2: 0.02,  # Minimum distance to YOLO_ID=2 (2% = almost touching!)
    # }

    physical_contact_frames: Dict[int, int]
    # E.g.: {
    #   1: 0,     # Never touched
    #   2: 15,    # Touched for 15 frames (talking, very close)
    # }

    # Spatial trajectory
    trajectory_divergence: Dict[int, float]
    # E.g.: {
    #   1: 0.9,   # 90% divergent paths (going different directions)
    #   2: 0.1,   # 10% divergent (moving together, close coordination)
    # }
```

**Why these fields?**
- Spatial proximity confirms if agents are interacting vs passing
- Diverging trajectories suggest independent agents (not same person)
- Contact/interaction frames mark when they interact (group activity)

#### 2.4 Track Relationship Metadata

```python
class LocalTrack:
    # NEW FIELDS: Relationships between concurrent tracks

    interaction_type: Optional[str]
    # Values: "none", "passing", "talking", "cooperating", "conflicting"
    # Determined by: distance, contact frames, relative motion

    inferred_relationship: Optional[Dict[int, str]]
    # E.g.: {
    #   1: "gardener",          # This track identified as Gardener
    #   2: "visitor_talking",   # Visitor engaged in conversation
    #   3: "passerby",          # Unknown person passing through
    # }

    # Track continuity hints
    likely_same_person_as: Optional[List[int]]
    # E.g.: [5, 8]  # This track probably continues as YOLO_ID=5 later (local_id 8 in next chunk)
    # Used when YOLO loses and reacquires ID
```

## 3. How This Enables Automatic Separation

### Before: Gardener + Visitor = GROUP (ambiguous)
```
T=0-5s: Gardener alone
Track-A (solo, clean for grid)

T=5-6s: Gardener + Visitor
YOLO says:
  - YOLO_ID=1 (appears in frames 150-200)
  - YOLO_ID=2 (appears in frames 150-200)

Previous design: Create GROUP track
  - Can't determine which is Gardener
  - Can't learn grid from this

Result: Ambiguity, requires LLM/manual resolution
```

### After: Automatic Separation
```
T=0-5s: Gardener alone
Track-A: agent_id=Gardener, solo=TRUE

T=5-6s: Both visible (but separate YOLO IDs!)
Track-A continues: agent_id=Gardener, solo=FALSE, max_concurrent=2
Track-V: agent_id=UNKNOWN, solo=FALSE, is_yolo_tracked=TRUE
  - Linked to Track-A as "concurrent_with"
  - Visual dissimilarity to Gardener (0.08)
  - Trajectory divergence (0.85)

T=6s: Visitor exits
Track-A resumes: solo=FALSE -> TRUE (transition back)
Track-V ends: solo=FALSE

Result: Two separate, unambiguous tracks
- Gardener's track is continuous (no identity questions)
- Visitor's track is isolated and clear
- Both tracks clean enough for grid learning!
```

## 4. Implementation: YOLO-Aware Track Creation

### Step 1: Parse YOLO Output with Confidence

```python
def create_local_track_from_yolo(yolo_tracking_output, chunk_id, camera_id):
    """
    Create LocalTrack from YOLO11 tracking results.
    Preserve YOLO confidence information.
    """
    for detection in yolo_tracking_output:
        yolo_local_id = detection.id  # YOLO's tracking ID

        # Get or create track
        if yolo_local_id not in chunk.local_tracks:
            local_track = LocalTrack(
                local_id=yolo_local_id,
                camera_id=camera_id,
                chunk_id=chunk_id,
                is_yolo_tracked=True,
                tracking_confidence=detection.tracking_confidence,  # NEW
                tracking_interruptions=0,  # Will update if ID lost/regained
            )
            chunk.local_tracks[yolo_local_id] = local_track

        # Add detection
        track = chunk.local_tracks[yolo_local_id]
        track.detections.append(detection)

        # Track if ID was lost (gap in frames)
        if len(track.detections) > 1:
            prev_frame = track.detections[-2].frame_number
            curr_frame = detection.frame_number
            if curr_frame - prev_frame > 1:
                track.tracking_interruptions += 1  # NEW
```

### Step 2: Compute Concurrent Agent Metrics

```python
def analyze_concurrent_agents(chunk: TrackChunk):
    """
    For each local track, analyze relationships with concurrent tracks.
    """
    local_tracks = list(chunk.local_tracks.values())

    for track_a in local_tracks:
        # Find tracks that overlap temporally
        concurrent = [
            t for t in local_tracks
            if t.local_id != track_a.local_id
            and frames_overlap(track_a.detections, t.detections)
        ]

        if not concurrent:
            track_a.concurrent_tracks = []
            continue

        track_a.concurrent_tracks = [t.local_id for t in concurrent]

        # For each concurrent track, compute metrics
        for track_b in concurrent:
            # Visual similarity
            visual_sim = compute_visual_similarity(
                track_a.detections,
                track_b.detections
            )
            track_a.visual_similarity_to_others[track_b.local_id] = visual_sim

            # Spatial metrics
            min_dist = compute_minimum_distance(track_a, track_b)
            contact_frames = count_touching_frames(track_a, track_b)
            divergence = compute_trajectory_divergence(track_a, track_b)

            track_a.min_distance_to_others[track_b.local_id] = min_dist
            track_a.physical_contact_frames[track_b.local_id] = contact_frames
            track_a.trajectory_divergence[track_b.local_id] = divergence

            # Infer interaction type
            if contact_frames > 20:  # >0.5s at 30fps
                interaction = "talking"
            elif min_dist < 0.1:  # Closer than 10% of frame width
                interaction = "passing"
            else:
                interaction = "none"

            if not track_a.interaction_type:
                track_a.interaction_type = interaction
```

### Step 3: Map LocalTrack to Global Agents

```python
def promote_local_tracks_to_global(chunk: TrackChunk, existing_agents: Dict[int, Agent]):
    """
    Convert YOLO local_id tracks to global agent tracks.
    KEY: Don't create ambiguous assignments. Use YOLO separation.
    """
    for local_track in chunk.local_tracks.values():
        # Step 1: Find cross-camera merge candidates
        merge_candidates = find_merge_candidates(local_track, existing_agents)

        # Step 2: IMPORTANT: Existing agent takes priority if unambiguous
        if local_track.local_id in previously_known_agents:
            # This YOLO_ID was assigned to an agent before
            agent = previously_known_agents[local_track.local_id]
            track = create_track(
                agent_id=agent.agent_id,
                camera=local_track.camera_id,
                start_time=local_track.start_time,
                end_time=local_track.end_time,
                solo_track=len(local_track.concurrent_tracks) == 0,
                max_concurrent_agents=len(local_track.concurrent_tracks) + 1,
                is_yolo_tracked=True,
                tracking_confidence=local_track.tracking_confidence
            )
            return track

        # Step 3: If new person, create as separate agent (no ambiguity!)
        if len(merge_candidates) == 0:
            # Completely new person
            new_agent = Agent(
                agent_id=generate_unknown_id(),
                agent_type=local_track.agent_type
            )
            track = create_track(
                agent_id=new_agent.agent_id,
                camera=local_track.camera_id,
                start_time=local_track.start_time,
                end_time=local_track.end_time,
                solo_track=len(local_track.concurrent_tracks) == 0,
                max_concurrent_agents=len(local_track.concurrent_tracks) + 1,
                is_yolo_tracked=True,
                tracking_confidence=local_track.tracking_confidence
            )
            return track

        # Step 4: Merge candidates exist. Can YOLO help disambiguate?
        if len(merge_candidates) == 1:
            # Only one possible match - clean!
            agent = merge_candidates[0]
            track = create_track(agent_id=agent.agent_id, ...)
            return track

        # Step 5: Multiple candidates. Does visual similarity help?
        if len(merge_candidates) > 1:
            # Use Re-ID + clothing similarity to pick best match
            best_match = rank_candidates_by_similarity(
                local_track.dominant_color,
                local_track.color_histogram,
                merge_candidates
            )

            if best_match.confidence > 0.8:
                # High confidence match
                track = create_track(agent_id=best_match.agent_id, ...)
                return track
            else:
                # Still ambiguous - mark as ambiguous
                track = create_track(
                    agent_id=None,
                    candidate_agents=[c.agent_id for c in merge_candidates],
                    solo_track=False,
                    is_yolo_tracked=True
                )
                return track
```

## 5. Why This Works Better

### Separation at Detection Level, Not Decision Level

**Old approach:**
```
Merge decision required ← Ambiguity
                           ↑
                      GROUP track
                           ↑
                      Detection (both visible)
```

**New approach:**
```
Unambiguous tracks → Grid learning enabled
                     ↑
                Detection + YOLO tracking
                (YOLO already separated them locally!)
```

### Concrete Example: Gardener + Visitor

**Old:**
```
Track-GROUP: {Gardener, Visitor}  ← Can't use for grid
  Solo=FALSE
  candidate_agents = [Agent-Gardener, Agent-Visitor]
  → Requires LLM review
  → Delays disambiguation
  → Visitor trajectory not learned
```

**New:**
```
Track-Gardener: agent_id=Gardener
  Solo=FALSE (had concurrent agents)
  is_yolo_tracked=TRUE
  tracking_confidence=0.98
  → Can be used for grid (single agent, clean signal)
  → No ambiguity

Track-Visitor: agent_id=Unknown-5
  Solo=FALSE (had concurrent agents)
  is_yolo_tracked=TRUE
  tracking_confidence=0.95
  visual_similarity_to_gardener=0.08  ← Clearly different!
  → Can be used for grid (single agent, clean signal)
  → No ambiguity

Result: BOTH tracks grid-learnable immediately
        Visitor's path automatically understood
        No manual review needed
```

## 6. Edge Cases and YOLO Limitations

### Case 1: YOLO Loses Track (Occlusion)

```
YOLO_ID=1: Gardener (frames 0-50)
           [leaves frame temporarily]
           Frames 51-80: [no detection]
YOLO_ID=3: "Re-acquired" at frame 81, but is it Gardener or new person?
```

**Solution:**
```python
if track.tracking_interruptions > 0:
    # After loss, lower confidence
    track.is_reacquired_after_loss = TRUE
    track.tracking_confidence *= 0.7  # Penalize

    # Use visual similarity to confirm
    if similarity_to_previous > 0.85:
        # Likely same person
        confidence *= 1.2
    else:
        # Different person - create new track
        new_track = create_track(...)
```

### Case 2: YOLO Assigns Same Person Multiple IDs (ID Drift)

```
YOLO_ID=1: Person walks across frame (frames 0-100)
YOLO_ID=2: Same person (YOLO lost ID at occlusion, reassigned)
```

**Solution:**
```python
# Look for YOLO_IDs that are highly similar AND spatially continuous
def detect_id_drift(chunk: TrackChunk):
    """Detect if YOLO assigned multiple IDs to same person."""
    local_tracks = list(chunk.local_tracks.values())

    for track_a, track_b in combinations(local_tracks):
        if visual_similarity(track_a, track_b) > 0.92:
            if trajectory_continuous(track_a, track_b):
                # Probably same person, different YOLO IDs
                # Merge the two local tracks
                merge_local_tracks(track_a, track_b)
```

### Case 3: Person Briefly Visible, Very Similar to Another

```
YOLO_ID=1: Gardener (continuous)
YOLO_ID=2: Visitor who looks very similar (0.78 similarity)

But YOLO says they're different, and spatial distance is 0.1m
→ Trust YOLO, they're different people
```

**Solution:**
```python
# Use spatial proximity as tiebreaker
if min_distance > 0.05:  # > 5% of frame apart
    # If far enough apart, trust YOLO's separate ID even if visually similar
    # (Might be twins, similar clothing, etc.)
    confidence_in_separation *= 1.5
```

## 7. Summary: Required Data Fields

Add to `LocalTrack` class:

```python
class LocalTrack:
    # YOLO tracking quality (NEW)
    tracking_confidence: float        # YOLO's confidence in continuous tracking
    tracking_interruptions: int       # How many times ID was lost/regained
    id_stability_score: float        # (frames_with_id / total_frames)
    is_yolo_tracked: bool            # TRUE if continuously tracked
    is_reacquired_after_loss: bool   # TRUE if ID lost then regained

    # Concurrent agent analysis (NEW)
    concurrent_tracks: List[int]     # YOLO IDs visible simultaneously
    visual_similarity_to_others: Dict[int, float]
    dominant_color: Tuple[int, int, int]
    color_histogram: bytes
    min_distance_to_others: Dict[int, float]
    physical_contact_frames: Dict[int, int]
    trajectory_divergence: Dict[int, float]
    interaction_type: Optional[str]  # "talking", "passing", "none"
    inferred_relationship: Optional[Dict[int, str]]
    likely_same_person_as: Optional[List[int]]
```

These fields enable:
1. ✅ Confident separation of concurrent agents (no GROUP track)
2. ✅ Both tracks usable for grid learning
3. ✅ Visitor's trajectory automatically tracked
4. ✅ No ambiguity, no LLM review needed for simple cases
5. ✅ GUI shows clean path: Visitor → Garden → Talks → Leaves

## 8. Related Documents

- **L13_Detection.md:** LocalTrack data structure and YOLO integration
- **L12_ConcurrentTracking.md:** Comparison with GROUP track approach (now superseded)
- **L12_Merging.md:** How separate tracks are merged across cameras
- **L12_TrackAgentRelationship.md:** Track-to-agent mapping (no ambiguity here!)
