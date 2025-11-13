# Agent Tracking & Merge Logic

This document details how the system tracks persistent agents (people, vehicles) across cameras, through occlusions, and into/out of containers (vehicles, structures). It uses a dual-grid system and observable-evidence-based merging.

## Core Philosophy

**Think like a security analyst watching footage blind:**
- No statistical ownership priors
- No routine assumptions
- Only what can be SEEN or INFERRED from visible evidence
- Honest about uncertainty (don't force wrong guesses)
- Self-correcting as more evidence emerges

**Codifiable and Observable:**
- Simple evidence checks, not weighted formulas
- Count evidence, don't do arithmetic on scores
- All decisions traceable to observable facts
- Defer complex uncertainty to Phase 4-5 (LLM with video)

## Where Confidence Scores Are (Legitimately) Used

The system uses confidence/uncertainty in THREE specific places:

1. **Agent Existence** (Phase 1-2): Is this detector output a real agent or noise?
   - Detector confidence + persistence filter → create agent or ignore
   
2. **Merge Decisions** (Phase 2-3): AUTO_MERGE vs REVIEW_LLM vs REVIEW_HUMAN?
   - Count strong/weak evidence → simple decision rules
   - NO score arithmetic (0.30 + 0.40 + 0.50)
   - YES simple thresholds (strong_evidence ≥ 2 → auto-merge)

3. **Location Uncertainty** (Phase 4-5, deferred): Where is agent when not visible?
   - `possible_locations[]` structure exists but implementation deferred
   - Human or LLM with video evidence can resolve deterministically
   - Needs observable clues (last seen location, vehicle departures, timing)
   - Will NOT use fractional confidence arithmetic

## Dual-Grid System

### 32×32 Intra-Camera Grid

**Purpose:** Track agent movement and state within a single camera frame.

**Design:**
```python
# Perspective-aware: larger cells at bottom/center, smaller at top/edges
# Auto-calibrated depth from observed person heights
grid_32x32[camera] = {
    'rows': 32,
    'cols': 32,
    'perspective': {
        'k_row': 1.6,  # Vertical perspective (taller bottom rows)
        'horiz_vanish_strength': 0.30,
        'bottom_expand_strength': 0.15,
        'col_center_emphasis': 0.20
    },
    'height_calibration': {}  # [row][col] → median_bbox_height (learned online)
}
```

**Tracking mechanics:**
- Detect person → map bbox centroid to cell (row, col)
- Track frame-to-frame at **pixel resolution** (bbox centroids)
- Grid provides **spatial consistency checks**:
  - Same cell = likely same person (suppress bbox jitter)
  - Adjacent cell = plausible continuation
  - Jump 10 cells = probably ID swap, flag "possibly_switched"

**Anti-swap logic (multi-person scenarios):**
```python
def check_id_swap(track_a, track_b, frame):
    """Detect when two people pass and IDs might have swapped"""
    
    # Spatial consistency: Did track jump unreasonably far?
    cell_distance = manhattan_distance(track_a.last_cell, track_a.current_cell)
    if cell_distance > 5:  # Threshold tunable
        track_a.possibly_switched = True
    
    # Motion vector consistency: Did direction change drastically?
    motion_angle_change = angle_between(track_a.prev_vector, track_a.current_vector)
    if motion_angle_change > 90:  # degrees
        track_a.possibly_switched = True
    
    # Cross-over detection: Did two tracks intersect?
    if tracks_intersected(track_a, track_b):
        # Both tracks flagged as uncertain at intersection point
        track_a.possibly_switched = True
        track_b.possibly_switched = True
        # Queue for review or use face/appearance to resolve
```

**Movement calculation:**
```python
# Accumulate pixel distance traveled
for frame in track.frames:
    pixel_delta = distance(frame.bbox_center, prev_frame.bbox_center)
    
    # Normalize by depth (using height calibration)
    cell = get_cell_32x32(frame.bbox_center)
    depth_factor = height_calibration[camera][cell]
    
    real_world_distance += pixel_delta * depth_factor

# Promote to Active if:
if real_world_distance > MOVEMENT_THRESHOLD and duration >= 3.0:
    camera.state = ACTIVE
```

### 8×8 Inter-Camera Grid

**Purpose:** Simplified representation for neighbor camera boost (Armed state propagation).

**Mapping:**
```python
# Clean 4:1 downsampling from 32×32
def to_8x8(cell_32x32):
    row_32, col_32 = cell_32x32
    return (row_32 // 4, col_32 // 4)

# Examples:
# 32×32 (0,0) → 8×8 (0,0)
# 32×32 (3,3) → 8×8 (0,0)
# 32×32 (31,31) → 8×8 (7,7)
```

**Portal Configuration (not learning):**

Portals are manually configured connections between cameras. See `PORTAL_CONFIGURATION.md` for details.

```yaml
# config/property_map.yaml
portals:
  - name: "Front Door"
    type: [person, vehicle]
    connections:
      - camera_a: "FrontPorch"
        cells_a: [[15,20], [16,20]]  # 32×32 cells
        camera_b: "Entryway"
        cells_b: [[3,5]]
        max_time: 15  # seconds (loose filter)
```

**What portals do:**
- Track ends on CamA in portal cell → look for candidates on CamB in portal cell
- Time window: Δt < max_time (default 60s, just a sanity check)
- Face match or LLM decides if same agent
- No statistics, no learning, just adjacency filter

## Agent Data Structure

```python
class Agent:
    """Persistent entity tracked across cameras and time"""
    
    def __init__(self, agent_type):
        self.id = uuid4()
        self.type = agent_type  # 'person' | 'vehicle'
        
        # Identity (initially unknown)
        self.identity = None  # 'Resident-1' | 'Unknown-47' | plate number
        
        # Visibility: Track segments where we can SEE the agent
        self.track_segments = []  # List of TrackSegment objects
        
        # Current state
        self.current_state = {
            'visible': True | False,
            'camera': 'CamC' | None,
            'cell_32x32': (15, 20) | None,
            'cell_8x8': (3, 5) | None,  # Derived via to_8x8()
            'last_seen': timestamp,
            'motion_vector': (dx, dy) | None,
            'possibly_switched': False  # Flag for uncertain ID
        }
        
        # When not visible, possible locations (Phase 4-5)
        # Structure exists for future LLM/video analysis
        # Human can deterministically resolve from observable clues
        self.possible_locations = []  # List of {state, location, evidence, reason}
        
        # Image bank for re-identification
        self.image_bank = {}  # [camera][lighting] → [top_3_images]
        
        # Merge history
        self.merge_decisions = []  # History of associations, evidence
        
        # For vehicles
        self.plate = None
        self.passengers = []  # List of person agent IDs

class TrackSegment:
    """One continuous visibility window on a single camera"""
    
    def __init__(self, camera, start_time):
        self.id = uuid4()
        self.camera = camera
        self.start_time = start_time
        self.end_time = None  # Set when segment ends
        
        # Spatial path
        self.entry_cell_32x32 = None
        self.exit_cell_32x32 = None
        self.path_cells = []  # Sequence of cells traversed
        
        # Motion statistics
        self.total_distance_pixels = 0
        self.total_distance_real = 0  # Using depth calibration
        self.avg_velocity = 0
        
        # Detection metadata
        self.detections = []  # Frame-by-frame bboxes, confidences
        self.avg_confidence = 0
        self.quality_score = 0
        
        # Best images for re-ID
        self.best_images = []  # Top K frames with quality scores
```

## Agent Merge Decision Pipeline

### Phase 1: Find Candidates via Portals

```python
def find_merge_candidates(track_segment_ended):
    """
    Track segment just ended on CamA.
    Find plausible continuations via configured portals.
    """
    
    candidates = []
    exit_cell = track_segment.exit_cell_32x32
    exit_time = track_segment.end_time
    exit_camera = track_segment.camera
    
    # Get configured portals for this camera and cell
    portals = get_portals_for_camera_cell(exit_camera, exit_cell, 
                                          track_segment.agent_class)
    
    for portal in portals:
        # Get the other side of portal
        for connection in portal.connections:
            if connection.camera_a == exit_camera:
                target_camera = connection.camera_b
                target_cells = connection.cells_b
                max_time = connection.max_time or 60
            elif connection.camera_b == exit_camera:
                target_camera = connection.camera_a
                target_cells = connection.cells_a
                max_time = connection.max_time or 60
            else:
                continue
            
            # Simple time window
            time_window = (exit_time, exit_time + max_time)
            
            # Find tracks starting on other side
            target_segments = query_track_starts(
                camera=target_camera,
                cells_32x32=target_cells,
                time_range=time_window,
                agent_class=track_segment.agent_class
            )
            
            for target_seg in target_segments:
                candidates.append({
                    'source': track_segment,
                    'target': target_seg,
                    'portal': portal.name,
                    'delta_t': target_seg.start_time - exit_time
                })
    
    # If no portals configured, fall back to wide discovery (Phase 2)
    if not portals:
        # Check all cameras within reasonable time (60s)
        all_cameras = get_all_cameras()
        for cam in all_cameras:
            if cam == exit_camera:
                continue
            
            target_segments = query_track_starts(
                camera=cam,
                time_range=(exit_time, exit_time + 60),
                agent_class=track_segment.agent_class
            )
            
            for target_seg in target_segments:
                candidates.append({
                    'source': track_segment,
                    'target': target_seg,
                    'portal': None,  # Discovery mode
                    'delta_t': target_seg.start_time - exit_time
                })
    
    return candidates
```

### Phase 2: Observable Evidence Scoring

```python
def score_merge_candidate(source, target, edge):
    """
    Score using ONLY observable evidence.
    No weighted formulas - use simple boolean checks and count evidence.
    """
    
    evidence = []
    strong_evidence_count = 0
    weak_evidence_count = 0
    has_veto = False
    
    # 1. TIMING (simple sanity check)
    delta_t = target.start_time - source.end_time
    
    if delta_t < 0:
        evidence.append(f'simultaneous (overlap)')
        strong_evidence_count += 1
    elif delta_t <= 60:
        evidence.append(f'timing_plausible (Δt={delta_t:.1f}s)')
        weak_evidence_count += 1
    else:
        # Too slow, probably different agent
        evidence.append(f'timing_implausible (Δt={delta_t:.1f}s)')
        has_veto = True
    
    # 2. SPATIAL CONTINUITY (cells already match via 8×8 gating)
    # Check if paths show smooth continuation at 32×32 level
    spatial_continuity = check_path_continuity_32x32(source, target)
    if spatial_continuity:
        weak_evidence_count += 1
        evidence.append('spatial_continuous')
    
    # 3. FACE MATCH (if available)
    if source.best_face and target.best_face:
        face_sim = compare_faces(source.best_face, target.best_face)
        if face_sim > 0.85:
            strong_evidence_count += 2  # Face match is strong evidence
            evidence.append(f'face_match (sim={face_sim:.2f})')
        elif face_sim < 0.60:
            has_veto = True
            evidence.append(f'face_mismatch (sim={face_sim:.2f})')
    
    # 4. APPEARANCE CONSISTENCY (clothing, height)
    if source.avg_bbox_height and target.avg_bbox_height:
        height_ratio = min(source.avg_bbox_height, target.avg_bbox_height) / \
                      max(source.avg_bbox_height, target.avg_bbox_height)
        if height_ratio > 0.80:
            weak_evidence_count += 1
            evidence.append('height_consistent')
    
    # 5. ONLY CANDIDATE (no ambiguity)
    if len(all_candidates) == 1:
        strong_evidence_count += 1
        evidence.append('only_candidate')
    
    # DECISION LOGIC (simple rules, no arithmetic)
    if has_veto:
        decision = 'REJECT'
    elif strong_evidence_count >= 2:
        decision = 'AUTO_MERGE'
    elif strong_evidence_count >= 1 and weak_evidence_count >= 1:
        decision = 'REVIEW_LLM'
    elif strong_evidence_count >= 1 or weak_evidence_count >= 2:
        decision = 'REVIEW_HUMAN'
    else:
        decision = 'UNCERTAIN'
    
    return decision, evidence
```

### Phase 3: Decision Logic

```python
def make_merge_decision(candidates):
    """
    Decide whether to merge, suggest, or mark uncertain.
    Uses simple evidence counting, not score arithmetic.
    """
    
    if not candidates:
        return None  # No plausible continuations, track ends
    
    # Evaluate all candidates
    decisions = [(score_merge_candidate(c['source'], c['target'], c['edge']), c) 
                 for c in candidates]
    
    # Priority: AUTO_MERGE > REVIEW_LLM > REVIEW_HUMAN > UNCERTAIN > REJECT
    priority = {'AUTO_MERGE': 4, 'REVIEW_LLM': 3, 'REVIEW_HUMAN': 2, 'UNCERTAIN': 1, 'REJECT': 0}
    decisions.sort(key=lambda x: priority[x[0][0]], reverse=True)
    
    best_decision, best_evidence = decisions[0][0]
    best_candidate = decisions[0][1]
    
    # AUTO-MERGE (strong evidence)
    if best_decision == 'AUTO_MERGE':
        merge_agents(best_candidate['source'].agent, 
                    best_candidate['target'].agent,
                    evidence=best_evidence,
                    auto_merged=True)
        
        return {'decision': 'MERGED', 'evidence': best_evidence}
    
    # REVIEW WITH LLM (medium confidence, clear candidates)
    elif best_decision == 'REVIEW_LLM' and len([d for d in decisions if d[0][0] != 'REJECT']) <= 3:
        queue_for_llm_review({
            'candidates': decisions[:3],  # Top 3
            'suggested': best_candidate,
            'evidence': best_evidence,
            'review_type': 'merge_decision'
        })
        
        return {'decision': 'QUEUED_LLM', 'evidence': best_evidence}
    
    # REVIEW WITH HUMAN (uncertain or too many candidates)
    else:
        # Mark source agent with possible_locations (Phase 4-5)
        source_agent = best_candidate['source'].agent
        source_agent.current_state['visible'] = False
        
        # Store possibilities for future resolution (LLM with video, or human)
        source_agent.possible_locations = [
            {
                'state': 'possibly_continued',
                'camera': c['target'].camera,
                'cell_8x8': to_8x8(c['target'].entry_cell_32x32),
                'evidence': evidence,
                'reason': decision
            }
            for (decision, evidence), c in decisions[:5] if decision != 'REJECT'
        ]
        
        return {'decision': 'UNCERTAIN', 'possibilities': len(decisions)}
```

### Phase 4: LLM Adjudication (Queued Cases)

```python
def llm_adjudicate(review_item):
    """
    For queued cases: build panels and ask LLM.
    Uses Gemini Flash 2.0 (~$0.0001 per call).
    """
    
    source = review_item['suggested']['source']
    target = review_item['suggested']['target']
    
    # Build comparison panels
    panel_a = build_panel(source.best_images)
    panel_b = build_panel(target.best_images)
    
    # Prompt
    prompt = f"""
You are verifying if two image sequences show the same person.
Decide based ONLY on visible features: clothing, build, gait, face (if visible).
Ignore background and camera differences.

Context:
- Person exited {source.camera} at cell ({source.exit_cell_32x32}) at {source.end_time}
- Person entered {target.camera} at cell ({target.entry_cell_32x32}) at {target.start_time}
- Time between: {delta_t:.1f} seconds

Respond in JSON:
{{
  "decision": "YES" | "NO" | "UNCERTAIN",
  "reasoning": "brief explanation"
}}
"""
    
    response = llm_call(prompt, images=[panel_a, panel_b], temperature=0)
    decision = parse_json(response)
    
    # Simple decision logic - no confidence thresholds
    if decision['decision'] == 'YES':
        merge_agents(source.agent, target.agent,
                    evidence=['llm_yes', decision['reasoning']],
                    auto_merged=False,
                    needs_audit=True)
        return 'MERGED'
    
    elif decision['decision'] == 'NO':
        # Don't merge, record negative evidence
        record_rejected_merge(source, target, decision['reasoning'])
        return 'REJECTED'
    
    else:
        # Still uncertain, escalate to human
        queue_for_human_review(review_item, llm_decision=decision)
        return 'ESCALATED'
```

## Multi-State Location Tracking

**Note: Phase 4-5 implementation.** Structure defined now for future LLM/video analysis. A human watching video can deterministically figure out "did person enter vehicle or stay in garage?" from observable clues. LLM with video evidence can do the same. Not needed for Phase 2-3 basic tracking.

### Location States

```python
# Agent can be in multiple possible states (Phase 4-5)
class LocationState:
    VISIBLE = 'visible'              # Currently detected on camera
    IN_STRUCTURE = 'in_structure'    # Inside building (not visible)
    IN_VEHICLE = 'in_vehicle'        # Inside vehicle
    LEFT_PROPERTY = 'left_property'  # Vehicle departed with agent inside
    UNKNOWN = 'unknown'              # Lost tracking, uncertain location

# Example: Agent after vehicle departure (2 people, 1 vehicle ambiguity)
# Phase 4-5: LLM with video evidence can resolve by analyzing:
# - Who was last near vehicle bay?
# - Timing of person movements vs vehicle departure
# - Face visible through windshield?
agent_a.possible_locations = [
    {'state': IN_VEHICLE, 'vehicle': 'V1', 'evidence': ['near_vehicle_bay', 'recent_entry']},
    {'state': IN_STRUCTURE, 'location': 'garage', 'evidence': ['far_from_vehicle']},
    {'state': UNKNOWN, 'evidence': ['insufficient_data']}
]
```

### Uncertainty Collapse

```python
def agent_observed_elsewhere(agent, new_camera, new_cell, timestamp):
    """
    Direct observation collapses uncertainty.
    Rules out incompatible states - simple logic, no confidence updates.
    Phase 4-5: More sophisticated reasoning about timing/plausibility.
    """
    
    # Update definite location
    agent.current_state = {
        'visible': True,
        'camera': new_camera,
        'cell_32x32': new_cell,
        'cell_8x8': to_8x8(new_cell),
        'last_seen': timestamp
    }
    
    # Rule out incompatible locations (Phase 4-5: more sophisticated logic)
    for possible in agent.possible_locations:
        if possible['state'] == IN_VEHICLE:
            # Person seen on foot → was NOT in vehicle
            vehicle = possible['vehicle']
            record_correction({
                'agent': agent.id,
                'vehicle': vehicle.id,
                'predicted': 'in_vehicle',
                'actual': 'on_foot',
                'timestamp': timestamp,
                'learn_from_this': True
            })
        
        elif possible['state'] == LEFT_PROPERTY:
            # Person still on property → didn't leave
            record_correction({
                'agent': agent.id,
                'predicted': 'left_property',
                'actual': 'still_on_property',
                'timestamp': timestamp
            })
    
    # Clear uncertainties
    agent.possible_locations = []
    
    # If this agent was possibly_switched, now confirmed
    agent.current_state['possibly_switched'] = False
```

## Vehicle Container Handling

### Person → Vehicle Association

```python
def vehicle_departs(vehicle, garage_portal, timestamp):
    """
    Vehicle exits garage. Determine which people (if any) are inside.
    """
    
    # Find people who entered garage before vehicle departure
    people_in_garage = [
        agent for agent in all_agents 
        if agent.type == 'person' 
        and agent.last_location_matches(garage_portal.structure)
        and agent.last_seen < timestamp
    ]
    
    if len(people_in_garage) == 0:
        # Empty vehicle
        return []
    
    elif len(people_in_garage) == 1:
        # EASY CASE: One person, auto-associate
        person = people_in_garage[0]
        person.current_state = {
            'visible': False,
            'state': 'in_vehicle',
            'vehicle': vehicle.id,
            'timestamp': timestamp
        }
        # Note: Phase 4-5 - possible_locations for LLM/video resolution if needed
        person.possible_locations = [{
            'state': 'in_vehicle',
            'vehicle': vehicle.id,
            'reason': 'only_person_in_garage'
        }]
        vehicle.passengers.append(person.id)
        
        return [{'person': person.id, 'evidence': ['only_person']}]
    
    else:
        # HARD CASE: Multiple people - use simple evidence checks
        # Phase 4-5: LLM with video evidence can resolve deterministically
        associations = []
        
        for person in people_in_garage:
            evidence = []
            strong_evidence = 0
            weak_evidence = 0
            
            # Recency: Last person to enter more likely
            time_since_entry = timestamp - person.last_seen
            if time_since_entry < 60:
                weak_evidence += 1
                evidence.append(f'recent_entry ({time_since_entry}s)')
            
            # Spatial: Was person near vehicle bay?
            if person.last_cell_near_vehicle_bay():
                weak_evidence += 1
                evidence.append('near_vehicle_bay')
            
            # Face in vehicle (if visible through windshield)
            if vehicle.face_detected:
                face_match = compare_faces(person.best_face, vehicle.face_detected)
                if face_match > 0.85:
                    strong_evidence += 1
                    evidence.append(f'face_in_vehicle (match={face_match:.2f})')
            
            # DECISION (simple rules)
            if strong_evidence >= 1:
                # Strong evidence: assign to vehicle
                person.current_state['visible'] = False
                person.current_state['state'] = 'in_vehicle'
                person.possible_locations = [{
                    'state': 'in_vehicle',
                    'vehicle': vehicle.id,
                    'evidence': evidence
                }]
                associations.append({'person': person.id, 'evidence': evidence})
            
            elif weak_evidence >= 2:
                # Weak evidence: queue for review (Phase 4-5: LLM with video)
                person.possible_locations = [
                    {'state': 'in_vehicle', 'vehicle': vehicle.id, 'evidence': evidence},
                    {'state': 'in_structure', 'location': 'garage', 'evidence': ['no_vehicle_evidence']}
                ]
                queue_for_review({
                    'type': 'vehicle_association',
                    'vehicle': vehicle.id,
                    'person': person.id,
                    'evidence': evidence
                })
            
            else:
                # Insufficient evidence: mark uncertain (Phase 4-5: needs video analysis)
                person.possible_locations = [
                    {'state': 'in_vehicle', 'vehicle': vehicle.id, 'evidence': evidence},
                    {'state': 'in_structure', 'location': 'garage', 'evidence': ['remained_in_garage']},
                    {'state': 'unknown', 'evidence': ['insufficient_data']}
                ]
        
        return associations
```

## Image Bank for Re-Identification

**Quality metric philosophy:** Keep it simple and codifiable. Use straightforward heuristics that are easy to understand, tune, and debug. Avoid complex metrics like Laplacian variance for sharpness detection—these are hard to tune per camera and add brittleness. If simple metrics (bbox area × detection confidence) don't work well enough, can add brightness/size checks later, but start minimal.

```python
class AgentImageBank:
    """
    Store best quality images per agent for later re-identification.
    Even if face recognition fails in real-time, this enables manual or
    batch re-ID days/weeks later.
    
    Quality score = bbox_area × detection_confidence (simple, works)
    """
    
    def __init__(self, agent_id):
        self.agent_id = agent_id
        self.images = {}  # [camera][lighting_condition] → [Image objects]
    
    def add_image(self, camera, lighting, image, quality_score):
        """
        Add image if it's better than existing ones.
        Keep top 3 per camera × lighting condition.
        
        quality_score = bbox_width × bbox_height × detection_confidence
        """
        key = (camera, lighting)  # e.g., ('CamA', 'daylight')
        
        if key not in self.images:
            self.images[key] = []
        
        self.images[key].append({
            'image_path': save_image(image),
            'timestamp': now(),
            'quality_score': quality_score,
            'bbox_size': image.shape,
            'detection_confidence': image.detection_conf,
            'brightness_mean': np.mean(image),
            'brightness_std': np.std(image)
        })
        
        # Keep only top 3
        # Quality score = simple heuristic: bbox_area × detection_confidence
        # Avoid complex metrics like Laplacian variance (hard to tune/debug)
        self.images[key].sort(key=lambda x: x['quality_score'], reverse=True)
        self.images[key] = self.images[key][:3]
    
    def get_best_for_comparison(self, camera=None):
        """
        Retrieve best overall image or best for specific camera.
        """
        if camera:
            candidates = [img for key, imgs in self.images.items() 
                         if key[0] == camera for img in imgs]
        else:
            candidates = [img for imgs in self.images.values() for img in imgs]
        
        if not candidates:
            return None
        
        # Return highest quality
        return max(candidates, key=lambda x: x['quality_score'])
```

## Learning from Corrections

```python
def record_merge_correction(original_decision, human_override, context):
    """
    When human overrides a merge decision, extract learning signals.
    """
    
    correction = {
        'timestamp': now(),
        'original_decision': original_decision,
        'human_override': human_override,
        'context': context,  # All observable evidence used
        'learn_from': True
    }
    
    # What to learn?
    if original_decision == 'MERGE' and human_override == 'REJECT':
        # False positive merge
        # Possible causes:
        # - Time window too wide
        # - Face threshold too permissive
        # - Spatial plausibility overestimated
        
        if 'face_match' in context['evidence']:
            # Face match was not sufficient
            # → Raise face match threshold slightly
            adjust_threshold('face_match_min', +0.02)
        
        if context['delta_t'] > context['edge']['typical_time'] * 1.5:
            # Time fit was marginal
            # → Tighten time windows for this edge
            tighten_edge_window(context['edge'], factor=0.9)
    
    elif original_decision == 'REJECT' and human_override == 'MERGE':
        # False negative (missed merge)
        # Possible causes:
        # - Evidence thresholds too strict
        # - Didn't consider all evidence
        
        # → Review evidence counting rules (Phase 3-4)
        # → Consider adding new evidence types if pattern emerges
        log_false_negative(context)
    
    elif original_decision == 'UNCERTAIN' and human_override in ['MERGE', 'REJECT']:
        # System was uncertain, human resolved
        # → This is expected, no adjustment needed
        # → But record the pattern for similar future cases
        record_resolution_pattern(context, human_override)
    
    store_correction(correction)
```

## Re-ID Per Camera (Phase 3-4 Enhancement)

### Concept: Per-Camera Visual Fingerprints

The key insight: **perspective stays consistent within a camera**, even though it varies dramatically across cameras. Rather than trying to match faces/clothing across wildly different viewpoints, build separate Re-ID databases per camera.

### Why This Works

**For vehicles:**
- Same car from same camera angle → highly consistent appearance
- Color, shape, distinctive features all stable
- Much faster than LPR, more reliable than global Re-ID

**For people:**
- Clothing changes, but within a session (hours/day) often stable
- Gait, body shape, typical paths consistent per person per camera
- Faces work better when perspective is consistent

### Architecture

```python
class ReIDDatabase:
    """
    Per-camera visual fingerprints for known agents.
    """
    
    def __init__(self, camera_id):
        self.camera_id = camera_id
        self.fingerprints = {}  # agent_id -> [ReIDFingerprint]
    
    class ReIDFingerprint:
        agent_id: str
        camera_id: str
        embedding: np.ndarray  # 512-dim Re-ID embedding
        conditions: dict  # {'lighting': 'day', 'clothing_variant': 'A'}
        quality_score: float
        sample_images: list  # Reference crops for audit
        created_ts: datetime
        last_seen_ts: datetime
        match_count: int  # How many times successfully matched
        false_positive_count: int  # Human corrections

# One database per camera
reid_dbs = {
    'FrontDoor': ReIDDatabase('FrontDoor'),
    'Driveway': ReIDDatabase('Driveway'),
    # ... etc
}
```

### When to Build Fingerprints

```python
def on_validated_merge(agent, source_track, dest_track):
    """
    After human/LLM confirms two tracks are same agent,
    extract Re-ID fingerprints from both track segments.
    """
    
    # Extract from source camera
    if source_track.camera not in agent.reid_fingerprints:
        agent.reid_fingerprints[source_track.camera] = []
    
    # Get best quality crops from track
    crops = get_quality_crops(source_track, 
                             min_quality=0.6, 
                             max_count=5,
                             diversity='clothing_region')  # Full body or torso
    
    for crop in crops:
        embedding = reid_model.encode(crop)
        
        fingerprint = ReIDFingerprint(
            agent_id=agent.id,
            camera_id=source_track.camera,
            embedding=embedding,
            conditions={
                'lighting': detect_lighting(crop),  # day/night/dusk
                'clothing_variant': None  # Computed on-demand via similarity check
            },
            quality_score=crop.quality,
            sample_images=[crop.path],
            created_ts=now(),
            last_seen_ts=source_track.end_ts,
            match_count=0,
            false_positive_count=0
        )
        
        # Check if this fingerprint is diverse (not redundant)
        existing_fps = reid_dbs[source_track.camera].get_fingerprints(agent.id)
        
        if existing_fps:
            # Compute similarity to all existing fingerprints
            similarities = [cosine_similarity(embedding, fp.embedding) 
                          for fp in existing_fps]
            
            # If too similar to all existing (redundant outfit), skip
            if all(sim >= 0.80 for sim in similarities):
                continue  # Don't store redundant fingerprint
            
            # If we already have 5 fingerprints, prune least-used before adding
            if len(existing_fps) >= 5:
                least_used = min(existing_fps, key=lambda fp: fp.match_count)
                reid_dbs[source_track.camera].remove_fingerprint(least_used)
        
        reid_dbs[source_track.camera].add_fingerprint(fingerprint)
    
    # Repeat for destination camera
    # ... same process
```

### When to Use Re-ID

```python
def score_merge_candidate_with_reid(source_track, dest_track):
    """
    Before calling LLM, check if Re-ID match gives us high confidence.
    """
    
    # Only useful if we have fingerprints for this agent on dest camera
    candidate_agents = get_plausible_agents(source_track)
    
    best_match = None
    best_score = 0.0
    
    for agent in candidate_agents:
        # Do we have Re-ID fingerprints for this agent on dest_camera?
        fingerprints = reid_dbs[dest_track.camera].get_fingerprints(agent.id)
        
        if not fingerprints:
            continue  # No prior for this agent on this camera
        
        # Extract Re-ID embedding from dest track
        dest_crops = get_quality_crops(dest_track, min_quality=0.5, max_count=3)
        
        for dest_crop in dest_crops:
            dest_embedding = reid_model.encode(dest_crop)
            
            # Compare against all stored fingerprints for this agent
            for fp in fingerprints:
                # Cosine similarity
                similarity = cosine_similarity(dest_embedding, fp.embedding)
                
                # Adjust for conditions match
                if fp.conditions['lighting'] == detect_lighting(dest_crop):
                    similarity += 0.05  # Bonus for same lighting
                
                if similarity > best_score:
                    best_score = similarity
                    best_match = (agent, fp)
    
    # Decision thresholds (TBD after calibration)
    if best_score >= 0.85:
        # High confidence Re-ID match
        # → Auto-merge without LLM
        return {
            'decision': 'MERGE',
            'confidence': best_score,
            'evidence': ['reid_match', best_match[1].agent_id],
            'method': 'reid_per_camera'
        }
    
    elif best_score >= 0.70:
        # Medium confidence Re-ID match
        # → Use as strong evidence, but still call LLM with lower priority
        return {
            'decision': 'SUGGEST_MERGE',
            'confidence': best_score,
            'evidence': ['reid_weak_match', best_match[1].agent_id],
            'method': 'reid_per_camera_weak'
        }
    
    else:
        # No useful Re-ID match
        # → Fall back to normal merge pipeline (face/LLM)
        return None
```

### Maintenance & Cleanup

```python
# Prune stale fingerprints
def prune_reid_database(camera_id, max_age_days=90):
    """
    Remove fingerprints not seen recently.
    Keep only agents with recent activity.
    """
    db = reid_dbs[camera_id]
    cutoff = now() - timedelta(days=max_age_days)
    
    for agent_id in list(db.fingerprints.keys()):
        fps = db.fingerprints[agent_id]
        
        # Remove stale fingerprints
        fps = [fp for fp in fps if fp.last_seen_ts > cutoff]
        
        # Keep max 5 diverse fingerprints per agent per camera (wardrobe rotation)
        if len(fps) > 5:
            fps = sorted(fps, key=lambda x: (x.match_count, x.quality_score), 
                        reverse=True)[:5]
        
        if fps:
            db.fingerprints[agent_id] = fps
        else:
            del db.fingerprints[agent_id]

# Update match statistics on corrections
def on_reid_match_corrected(fingerprint, was_correct):
    """
    When human corrects a Re-ID match, update statistics.
    """
    if was_correct:
        fingerprint.match_count += 1
    else:
        fingerprint.false_positive_count += 1
        
        # If too many false positives, demote or remove
        false_rate = fingerprint.false_positive_count / max(1, fingerprint.match_count + fingerprint.false_positive_count)
        
        if false_rate > 0.3:
            # This fingerprint is unreliable, remove it
            reid_dbs[fingerprint.camera_id].remove_fingerprint(fingerprint)
```

### Performance Considerations

**Cost comparison (estimated):**
- Face detection + embedding: ~15-30ms per crop
- Re-ID embedding: ~5-10ms per crop
- Cosine similarity × 20 fingerprints: ~0.1ms
- **Total Re-ID check: ~10-20ms vs LLM call at ~200-500ms**

**Efficiency gains:**
- **Vehicles:** Re-ID likely 90%+ accurate on same camera, orders of magnitude faster than LPR
- **People:** Re-ID useful within session (same clothing), less useful across days
- **Cold start:** Requires validated merges to build database, so Phase 3-4 rollout

### Storage

```python
# Per-camera Re-ID database
storage/reid/{camera_id}/
    fingerprints.db          # SQLite with embeddings + metadata
    samples/{agent_id}/      # Reference image crops
        {fingerprint_id}_001.jpg
        {fingerprint_id}_002.jpg
        ...

# Schema
CREATE TABLE reid_fingerprints (
    fingerprint_id TEXT PRIMARY KEY,
    agent_id TEXT NOT NULL,
    camera_id TEXT NOT NULL,
    embedding BLOB NOT NULL,  -- 512-dim float32 array
    lighting TEXT,            -- 'day', 'night', 'dusk'
    clothing_variant TEXT,    -- 'A', 'B', 'C' (simple clustering)
    quality_score REAL,
    created_ts REAL,
    last_seen_ts REAL,
    match_count INTEGER DEFAULT 0,
    false_positive_count INTEGER DEFAULT 0,
    INDEX idx_agent_camera (agent_id, camera_id)
);
```

### Rollout Plan

**Phase 3 (Build Database):**
- After validated merges, extract Re-ID fingerprints from both tracks
- Store per camera × agent
- No active use yet, just accumulation

**Phase 4 (Soft Launch):**
- Check Re-ID before calling LLM
- If high confidence match (≥0.85), auto-merge
- Log all Re-ID decisions for calibration
- Human corrections update match statistics

**Phase 5 (Maturity):**
- Re-ID as primary method for vehicles on same camera
- Re-ID as fast pre-filter for people (within session)
- LLM only for low Re-ID confidence or cross-session
- Automatic pruning of stale/unreliable fingerprints

### Open Questions (TBD)

- **Re-ID model:** Which architecture? (OSNet, FastReID, custom fine-tune?)
- **Threshold calibration:** 0.85 auto-merge, 0.70 suggest — validate on real data
- **Clothing variant clustering:** Simple k-means on color histogram, or more sophisticated?
- **Vehicle specifics:** Separate model for vehicles vs people, or unified?
- **Cross-session persistence:** How long to keep fingerprints for infrequent visitors?

## Summary: Key Principles

1. **Dual grids:** 32×32 for intra-camera tracking, 8×8 for inter-camera learning (4:1 clean mapping)
2. **Agent persistence:** Agents exist conceptually even when not visible
3. **Observable evidence only:** No ownership priors, no routine assumptions
4. **Honest uncertainty:** Multi-state location tracking when ambiguous
5. **Learning flywheel:** Corrections strengthen time/space model over time
6. **Graceful degradation:** Switching agents within frame not catastrophic, flag "possibly_switched"
7. **Image banking:** Save diverse quality images for later re-ID
8. **LLM as tool:** Cheap adjudication for medium-confidence cases
9. **Re-ID per camera:** Build visual fingerprints where perspective is consistent (Phase 3-4)
10. **Self-correcting:** Later observations collapse uncertainties

---

This approach balances robustness (don't force wrong merges) with automation (learn to handle 90%+ cases without human input).
