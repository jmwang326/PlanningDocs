# Level 12 - Merging Technical Specs

## Purpose
Technical implementation of cross-camera merge logic.

## Architecture: Evidence Engine + Merging Engine

**Separation of Concerns:**
- **Evidence Engine:** Gathers all facts and evidence into a report
- **Track Merging Engine:** Makes decisions based on the evidence report

This document focuses on the **Merging Engine's decision logic**. The Evidence Engine is documented in L3.

---

## Merge Algorithm

### Candidate Finding
**Runs periodically (every 30s):**
```python
def find_merge_candidates():
    recent_tracks = get_tracks(last_60_seconds)
    for track1 in recent_tracks:
        for track2 in recent_tracks:
            if time_space_plausible(track1, track2):
                if portal_connected(track1.camera, track2.camera):
                    yield (track1, track2)
```

### Time/Space Gating
**Filters impossible transitions:**
- Temporal: `track2.start_time - track1.end_time < max_window`
- Spatial: distance / time_delta within walking speed (~1 m/s)
- Portal check: track endpoints near portal locations

### Evidence Scoring (Grid-Driven Hierarchy - Track Merging Engine)

**Core insight:** Grid learned from validated merges → enables deterministic merging of new tracks

```python
def evaluate_merge_candidate(track1, track2):
    """
    Evidence hierarchy: face → grid overlap → grid transition + alibi → LLM → reject
    """
    # 1. Face recognition (definitive)
    if face_match(track1, track2) > 0.75:
        return AUTO_MERGE, "face_match"

    # 2. GRID OVERLAP (fastest auto-merge)
    # Both tracks in same cell (or adjacent) with travel_time < 0.4s?
    # → Same camera can see both objects = same agent

    grid_stats = get_grid_transition_stats(
        track1.camera, track1.exit_cell,    # Where person left camera 1
        track2.camera, track2.entry_cell    # Where person entered camera 2
    )

    OVERLAP_THRESHOLD = 0.4  # seconds (tighter threshold for true overlaps)

    if grid_stats and grid_stats.travel_time < OVERLAP_THRESHOLD:
        # No time gap → cameras see same physical space
        # But: multiple people might be in that space

        # Count agents of same type visible in these cells at these times
        agents_at_exit = count_agents(track1.camera, track1.exit_cell, track1.end_time)
        agents_at_entry = count_agents(track2.camera, track2.entry_cell, track2.start_time)

        if agents_at_exit == 1 and agents_at_entry == 1:
            # Only one person visible in exit cell AND only one in entry cell
            # They must be the same person
            return AUTO_MERGE, "grid_overlap_single_agent"
        else:
            # Multiple people in overlap zone → ambiguous
            return MULTI_ASSIGN_OVERLAP, "grid_overlap_ambiguous"

    # 3. GRID TRANSITION (learned path: plausible but needs alibi check)
    # Learned travel time: 5 seconds typical. Observed: 4.8 seconds.
    # → Plausible path *from track1's camera*, but could entry agent come from elsewhere?

    if grid_stats and grid_stats.travel_time >= OVERLAP_THRESHOLD:
        travel_time_observed = track2.start_time - track1.end_time
        margin = max(2.0, grid_stats.variance * 2)  # Conservative margin

        if abs(travel_time_observed - grid_stats.travel_time) <= margin:
            # Time fits learned transition from track1 → track2

            # CRITICAL: Alibi check - could track2.entry have come from another camera?
            alternative_sources = find_alternative_sources_for_entry(
                track2.camera,
                track2.entry_cell,
                track2.start_time,
                master_grid=grid,
                exclude_camera=track1.camera
            )

            if not alternative_sources:
                # No plausible alternative sources
                # track1's camera is the only source that fits the grid + timing

                # Single agent confirmation at both ends
                agents_at_exit = count_agents(track1.camera, track1.exit_cell, track1.end_time)
                agents_at_entry = count_agents(track2.camera, track2.entry_cell, track2.start_time)

                if agents_at_exit == 1 and agents_at_entry == 1:
                    # Process of elimination: only track1 could be track2
                    return AUTO_MERGE, "grid_transition_single_source"
                else:
                    # Multiple agents → need additional evidence
                    return REVIEW_LLM, "grid_transition_multi_agent"
            else:
                # Multiple plausible sources (ambiguous)
                return REVIEW_LLM, "grid_transition_ambiguous_source"
        else:
            # Time doesn't match learned paths
            return REJECT, "grid_transition_timing_mismatch"

    # 4. NO GRID DATA (unlearned path or first observation)
    if not grid_stats:
        # Try manual portal configuration
        if manual_portal_exists(track1.camera, track2.camera):
            return REVIEW_LLM, "manual_portal_unlearned"
        else:
            # No evidence: learned grid, manual portal, or face
            return REJECT, "no_adjacency_evidence"

    # 5. Default: insufficient evidence
    return REJECT, "no_merge_evidence"

def find_alternative_sources_for_entry(dest_camera, dest_entry_cell, entry_time, master_grid, exclude_camera=None):
    """
    FAST ALIBI CHECK: Can the entry agent plausibly come from another camera?

    Uses fast lookup matrix for O(N) check instead of searching all 6×6×6×6 grids.

    Returns: List of source camera IDs that could plausibly be the source

    Algorithm:
    1. For each camera except exclude_camera:
        - Check fastest_inter_camera_time[(source_camera, dest_camera)]
        - If minimum time is unlearned, skip (no evidence)
        - If minimum time fits timing window, add to alternatives
    2. Return list of plausible source cameras

    A "plausible time window" means:
    - time_since_entry = entry_time
    - Could exit have happened from source_camera such that:
        - fastest_time <= (entry_time - exit_time) <= fastest_time + margin
    - Margin is conservative (e.g., 2-4 seconds) to account for variance
    """
    alternatives = []
    MARGIN = 3.0  # seconds - conservative allowance for variance

    for source_camera in get_all_cameras():
        if exclude_camera is not None and source_camera == exclude_camera:
            continue

        # Quick check: is there ANY learned path from this camera to dest?
        key = (source_camera, dest_camera)
        min_time = master_grid.fastest_inter_camera_time.get(key, None)

        if min_time is None:
            # No learned path - skip this source
            continue

        # Would timing fit?
        # Find tracks from source_camera that exited around entry_time - min_time
        expected_exit_time = entry_time - min_time
        exit_time_window = (expected_exit_time - MARGIN, expected_exit_time + MARGIN)

        candidate_tracks = get_tracks_exiting_camera_in_timeframe(
            source_camera,
            exit_time_window
        )

        if candidate_tracks:
            # At least one track could plausibly be the source
            alternatives.append(source_camera)

    return alternatives
```

**Key decision points:**
- **Grid < 0.4s + single agent in each zone** = instant auto-merge (temporal and spatial determinism)
- **Grid ≥ 0.4s + time fits + NO alternatives + single agents** = auto-merge (process of elimination)
- **Grid ≥ 0.4s + time fits + multiple alternatives** = REVIEW_LLM (ambiguous source)
- **Grid not learned** = need manual portal or high face confidence
- **Multiple agents in zone** = mark as ambiguous, needs additional evidence

## Face Recognition

### CodeProject.AI Integration
**Endpoint:**
```
POST http://{codeproject_server}/v1/vision/face/recognize
```

**Request:**
```json
{
  "image": "base64_encoded_jpeg",
  "min_confidence": 0.60
}
```

**Response:**
```json
{
  "predictions": [
    {
      "userid": "person_a",
      "confidence": 0.85
    }
  ]
}
```

### Face Library Management
- Register face: POST to `/v1/vision/face/register`
- Delete face: DELETE `/v1/vision/face/{userid}`
- List faces: GET `/v1/vision/face/list`

### Face Library Population Strategy

**Goal:** 2 faces per camera per lighting condition per person

**Rationale:**
- Lighting varies by time of day (morning, afternoon, evening, night)
- Camera angles vary across property
- Diverse library improves recognition accuracy

**Data Structure:**
```python
class FaceLibrary:
    """
    Face database for person agent
    """
    agent_id: int
    user_id: str  # CodeProject.AI userid

    faces: List[RegisteredFace]

class RegisteredFace:
    """
    Single face registration entry
    """
    face_crop: bytes           # Face-only crop (upscaled if needed)
    camera_id: int             # Which camera captured this
    lighting_condition: str    # "morning", "afternoon", "evening", "night"
    timestamp: float           # When captured
    quality_score: float       # Face quality metric
    source_track_id: int       # Track this came from (for audit)
```

**Lighting Classification:**
```python
def classify_lighting(timestamp: float, brightness: float) -> str:
    """
    Classify lighting condition for face registration diversity

    Uses timestamp (time of day) and frame brightness
    """
    hour = datetime.fromtimestamp(timestamp).hour

    # Time-based classification
    if 6 <= hour < 11:
        base_condition = "morning"
    elif 11 <= hour < 17:
        base_condition = "afternoon"
    elif 17 <= hour < 21:
        base_condition = "evening"
    else:
        base_condition = "night"

    # Adjust for actual brightness (handles overcast, artificial light)
    if brightness < 80:
        base_condition += "_low_light"

    return base_condition
```

**Population Workflow:**
```python
def populate_face_library_from_validated_merge(agent_id: int, track_a: LocalTrack, track_b: LocalTrack):
    """
    After LLM/human validates merge, add faces to library

    Target: 2 faces per camera per lighting condition
    """
    library = get_face_library(agent_id)

    # Process both tracks
    for track in [track_a, track_b]:
        # Check if we need more faces from this camera/lighting combo
        lighting = classify_lighting(track.start_time, track.avg_brightness)

        existing = [
            f for f in library.faces
            if f.camera_id == track.camera_id and f.lighting_condition == lighting
        ]

        needed = max(0, 2 - len(existing))

        if needed > 0:
            # Select best face frames from track
            candidates = [d for d in track.detections if d.face_visible and d.face_crop]
            candidates.sort(key=lambda d: d.face_quality, reverse=True)

            # Register top N needed
            for detection in candidates[:needed]:
                register_face(
                    user_id=library.user_id,
                    face_crop=detection.face_crop,
                    camera_id=track.camera_id,
                    lighting_condition=lighting,
                    quality_score=detection.face_quality
                )
```

**Registration to CodeProject.AI:**
```python
def register_face(user_id: str, face_crop: bytes, camera_id: int, lighting_condition: str, quality_score: float):
    """
    Register face crop to CodeProject.AI face library
    """
    # POST to CodeProject.AI
    response = requests.post(
        f"http://{codeproject_server}/v1/vision/face/register",
        data={
            "userid": user_id,
            "image": base64.b64encode(face_crop).decode()
        }
    )

    if response.status_code == 200:
        # Store metadata locally
        library = get_face_library_by_userid(user_id)
        library.faces.append(RegisteredFace(
            face_crop=face_crop,
            camera_id=camera_id,
            lighting_condition=lighting_condition,
            timestamp=time.time(),
            quality_score=quality_score
        ))
```

## Vehicle Re-ID Database

**Goal:** Appearance-based vehicle matching (similar to face recognition)

**Rationale:**
- Vehicles don't commit crimes (more liberal linking acceptable)
- Re-ID complements face recognition for vehicle-only scenarios
- Multiple orientations improve matching (front, side, rear)

**Data Structure:**
```python
class VehicleReIDLibrary:
    """
    Re-ID database for vehicle agent
    """
    agent_id: int
    vehicle_id: str  # Internal vehicle identifier

    appearances: List[VehicleAppearance]

class VehicleAppearance:
    """
    Single vehicle appearance entry (orientation + camera)
    """
    body_crop: bytes           # Full vehicle crop
    camera_id: int             # Which camera
    orientation: str           # "front", "side", "rear", "oblique"
    timestamp: float
    quality_score: float
    source_track_id: int
```

**Orientation Classification:**
```python
def classify_vehicle_orientation(bbox: BoundingBox, direction_vector: Tuple[float, float]) -> str:
    """
    Classify vehicle orientation from bbox aspect ratio and movement direction

    Orientations:
    - "front": moving toward camera
    - "rear": moving away from camera
    - "side": perpendicular to camera
    - "oblique": angled
    """
    aspect_ratio = bbox.width / bbox.height

    # Side view tends to be wider
    if aspect_ratio > 1.5:
        return "side"

    # Use movement direction to distinguish front/rear
    dx, dy = direction_vector
    angle = math.atan2(dy, dx)

    if -math.pi/4 < angle < math.pi/4:
        return "front"
    elif 3*math.pi/4 < abs(angle):
        return "rear"
    else:
        return "oblique"
```

**Population Workflow:**
```python
def populate_vehicle_reid_from_track(agent_id: int, track: LocalTrack):
    """
    Add vehicle appearances to Re-ID database

    Target: Multiple orientations across all cameras
    """
    library = get_vehicle_reid_library(agent_id)

    # Classify orientation
    orientation = classify_vehicle_orientation(
        track.bounding_box_at_midpoint(),
        track.direction_vector()
    )

    # Check if we have this camera/orientation combo
    existing = [
        a for a in library.appearances
        if a.camera_id == track.camera_id and a.orientation == orientation
    ]

    if len(existing) < 2:  # Target: 2 per camera/orientation
        # Select best frame
        candidates = sorted(track.detections, key=lambda d: d.quality_score, reverse=True)

        for detection in candidates[:1]:  # Add 1 at a time
            library.appearances.append(VehicleAppearance(
                body_crop=detection.body_crop,
                camera_id=track.camera_id,
                orientation=orientation,
                timestamp=detection.timestamp,
                quality_score=detection.quality_score,
                source_track_id=track.id
            ))
```

**Re-ID Matching (Placeholder for Future):**
```python
def vehicle_reid_match(track1: LocalTrack, track2: LocalTrack) -> float:
    """
    Appearance-based vehicle matching (future implementation)

    Returns: similarity score (0.0-1.0)

    Note: Re-ID model TBD (osnet_x1_0, resnet50_ibn, etc.)
    Liberal matching threshold (e.g., 0.6 vs 0.75 for faces)
    """
    # TODO: Implement Re-ID model inference
    # For now, placeholder
    return 0.0
```

## Identity Management

A robust identity management system is critical for the human-in-the-loop workflow and for building the face/vehicle recognition libraries. This system ensures that agents are named consistently and prevents the creation of duplicate identities.

### 1. Central Identity Registry
- There must be a single, central database table that serves as the source of truth for all named identities (both people and vehicles).
- This registry links a unique, stable `agent_id` to a human-readable name (e.g., "John Doe", "UPS Truck").
- It prevents name normalization issues, ensuring "John Doe" and "john doe" are treated as the same entity. All names should be stored in a normalized format (e.g., lowercase, trimmed whitespace).

### 2. Manual Review UI: The Identity Picker
When a human operator is reviewing a merge and needs to assign or confirm an identity, the UI must present an "Identity Picker" with the following features:

- **Dropdown of Recent/Common Names:** The picker should be pre-populated with a short list of the most recently or frequently assigned names to speed up the workflow for common visitors.
- **Search Box with Fuzzy Matching:** A search box must be provided to find any existing identity in the registry. This search should be case-insensitive and handle minor misspellings or variations to actively warn the user if they are about to create a duplicate (e.g., typing "Jon Doe" should suggest "Did you mean John Doe?").
- **"New Identity" Button:** A clear button or option to create a new person or vehicle identity must be available for first-time appearances. This action should only be taken after the search has confirmed the identity does not already exist.

### 3. Workflow
1. When a user validates a merge for an unknown agent, the Identity Picker appears.
2. The user first uses the search box to ensure the agent doesn't already exist.
3. If the agent exists, the user selects it from the search results or dropdown.
4. If the agent is truly new, the user clicks "New Identity," provides a name, and the new identity is created in the central registry.
5. The validated face crops or vehicle appearances from the merge are then associated with this definitive `agent_id` in their respective libraries.

---

## LLM Adjudication

### Gemini Flash 2.0
**Endpoint:** Google AI Studio API

**Prompt template:**
```
Are these the same person/vehicle?

Image Set A:
{panel_a_images}

Image Set B:
{panel_b_images}

Context:
- Camera A → Camera B
- Time gap: {time_delta} seconds
- Portal: {portal_name}

Respond in JSON:
{
  "answer": "YES|NO|CANNOT",
  "confidence": 0.0-1.0,
  "reasons": "brief explanation"
}
```

### Panel Assembly

**2×3 layout (6 images total):**
- Top row: Track A best frames (K=3)
- Bottom row: Track B best frames (K=3)
- Image dimensions: 768×768 pixels total panel
- Uses full body crops for visual context

**Dual Crop Strategy:**
```python
class Detection:
    """
    Each detection stores both body and face crops
    """
    bbox: BoundingBox           # Full detection bounding box
    body_crop: bytes            # Full body JPEG crop (for LLM panels)
    face_crop: Optional[bytes]  # Face-only crop (for face recognition)
    face_bbox: Optional[BoundingBox]  # Face location within body crop

    # Quality metrics
    blur_score: float           # Lower is sharper (Laplacian variance)
    brightness: float           # Mean luminance (0-255)
    face_visible: bool          # Face detector found face
    face_quality: Optional[float]  # Face-specific quality score
```

**Why dual crops:**
- Body crops provide visual context for LLM/human comparison
- Face crops optimize face recognition (upscaling yields different results)
- Both stored to support different use cases without reprocessing

**Frame Selection Algorithm:**
```python
def select_best_frames(track: LocalTrack, K: int = 3) -> List[Detection]:
    """
    Select K best frames from track for panel assembly

    Prioritizes:
    1. Face visibility (face_visible = True)
    2. Sharpness (low blur_score)
    3. Brightness (mid-range, avoid over/underexposed)
    4. Temporal diversity (spread across track duration)
    """
    candidates = [d for d in track.detections if d.face_visible]

    if len(candidates) < K:
        # Fallback to body-only frames if insufficient face frames
        candidates = track.detections

    # Score each frame
    scored = []
    for detection in candidates:
        score = compute_quality_score(detection)
        scored.append((score, detection))

    # Sort by quality, take top K
    scored.sort(key=lambda x: x[0], reverse=True)
    best = [d for _, d in scored[:K]]

    # Ensure temporal diversity (avoid clustering)
    best = ensure_temporal_diversity(best, track.duration)

    return best

def compute_quality_score(detection: Detection) -> float:
    """
    Composite quality score for frame selection
    """
    # Sharpness component (normalize blur to 0-1, invert)
    sharpness = 1.0 / (1.0 + detection.blur_score / 100.0)

    # Brightness component (penalize extremes)
    brightness_ideal = 128.0
    brightness_penalty = abs(detection.brightness - brightness_ideal) / brightness_ideal
    brightness_score = 1.0 - brightness_penalty

    # Face visibility bonus
    face_bonus = 1.5 if detection.face_visible else 1.0

    # Face quality (if available)
    face_quality = detection.face_quality if detection.face_quality else 0.5

    # Weighted combination
    score = (
        0.4 * sharpness +
        0.2 * brightness_score +
        0.4 * face_quality
    ) * face_bonus

    return score
```

**Panel Assembly Pipeline:**
```python
def assemble_comparison_panel(track_a: LocalTrack, track_b: LocalTrack) -> bytes:
    """
    Create 2×3 comparison panel for LLM adjudication

    Returns: JPEG image (768×768 pixels)
    """
    # Select best frames
    frames_a = select_best_frames(track_a, K=3)
    frames_b = select_best_frames(track_b, K=3)

    # Use body crops (full context for LLM)
    crops_a = [f.body_crop for f in frames_a]
    crops_b = [f.body_crop for f in frames_b]

    # Assemble grid
    panel = create_image_grid(
        top_row=crops_a,
        bottom_row=crops_b,
        output_size=(768, 768),
        add_labels=True  # "Track A - Camera 1", "Track B - Camera 4"
    )

    return panel
```

## Inter-Camera Grid (6×6 + Fast Lookup)

### Data Structure
```python
class InterCameraGrid:
    """
    6×6 grid per camera pair for learning spatial adjacency.

    Two-tier storage:
    1. Detailed 6×6×6×6 grids: full cell-to-cell transition statistics
    2. Fast lookup matrix: minimum time ever observed per camera-pair
    """
    # DETAILED: Full 6×6×6×6 grid per camera pair
    # Key: (source_camera_id, dest_camera_id)
    # Value: 6×6 source cells → 6×6 dest cells → TransitionStats
    transitions: Dict[Tuple[int, int], Dict[int, Dict[int, TransitionStats]]]
    # transitions[(src_cam, dst_cam)][src_cell][dst_cell] = TransitionStats

    # FAST LOOKUP: Minimum time from any source to destination camera
    # Key: (source_camera_id, dest_camera_id)
    # Value: minimum observed travel time in seconds (or None if unlearned)
    fastest_inter_camera_time: Dict[Tuple[int, int], Optional[float]]
    # fastest_inter_camera_time[(src_cam, dst_cam)] = min_travel_time

class TransitionStats:
    """
    Learned statistics for cell pair transitions
    """
    typical_time: float  # Median transition time (seconds)
    variance: float      # Variance in transition times
    sample_count: int    # Number of validated transitions
    last_updated: float  # Timestamp

    OVERLAP_THRESHOLD = 0.4  # seconds (cameras see same physical space)
    PORTAL_MIN_TIME = 0.4    # seconds (anything >= overlap threshold)
    PORTAL_MAX_TIME = 15.0   # seconds (reasonable portal crossing time)

    def is_overlap(self) -> bool:
        """Overlap if typical time < 0.4s (cameras see same physical space)"""
        return self.typical_time < self.OVERLAP_THRESHOLD

    def is_portal(self) -> bool:
        """Portal if 0.4-15s transition time"""
        return self.PORTAL_MIN_TIME <= self.typical_time <= self.PORTAL_MAX_TIME

    def update(self, time_delta: float):
        """Update running statistics with new observation"""
        # Running median (robust to outliers)
        # Exponential decay for variance
        pass  # Implementation details

def position_to_grid_cell(position: Tuple[float, float], grid_size: int = 6) -> int:
    """
    Convert normalized position (0-1, 0-1) to grid cell index

    Returns: cell_id (0-35 for 6×6 grid)
    """
    x, y = position
    col = int(x * grid_size)
    row = int(y * grid_size)
    return row * grid_size + col
```

### Bootstrap Learning
```python
def learn_from_validated_merge(track_a, track_b, master_grid):
    """
    User/LLM validated: Track A and Track B are same agent
    Update both detailed grid and fast lookup matrix.
    """
    cell_a = position_to_grid_cell(track_a.end_position)
    cell_b = position_to_grid_cell(track_b.start_position)

    time_delta = track_b.start_time - track_a.end_time

    # ===== UPDATE DETAILED 6×6×6×6 GRID =====
    # Get or create transition stats
    key = (track_a.camera, track_b.camera)
    if key not in master_grid.transitions:
        master_grid.transitions[key] = {}
    if cell_a not in master_grid.transitions[key]:
        master_grid.transitions[key][cell_a] = {}
    if cell_b not in master_grid.transitions[key][cell_a]:
        master_grid.transitions[key][cell_a][cell_b] = TransitionStats()

    # Update statistics
    stats = master_grid.transitions[key][cell_a][cell_b]
    stats.update(time_delta)

    # ===== UPDATE FAST LOOKUP MATRIX =====
    # Keep track of minimum time ever observed for this camera pair
    if key not in master_grid.fastest_inter_camera_time:
        master_grid.fastest_inter_camera_time[key] = time_delta
    else:
        # Update with minimum observed time
        current_min = master_grid.fastest_inter_camera_time[key]
        if current_min is None or time_delta < current_min:
            master_grid.fastest_inter_camera_time[key] = time_delta

    # Grid now knows:
    # - If time_delta ≈ 0: cameras overlap at these cells
    # - If time_delta 0.4-15s: portal connection
    # - If time_delta > 15s: distant cameras, rare transition
    # - fastest_inter_camera_time[(A→B)] = minimum time ever observed
```

## Multi-Assignment

### Data Structure
**Track can link to multiple agents:**
```python
class Track:
    agent_assignments: List[AgentAssignment]

class AgentAssignment:
    agent_id: int
    evidence: List[str]  # ["face_match", "portal_crossing", "grid_overlap_multi"]
    validated: bool
    # Note: No probabilities (see L2D_Strategy - no percentages)
```

### Uncertainty Persistence
- Assignments persist until validated (human/LLM)
- Timeline queries return all possibilities
- UI shows uncertainty ("possibly Agent A or B")

## Vehicle Association Technical Specs

### Agent State Structure
**Person agent:**
```python
class PersonAgent:
    agent_id: int
    state: Literal[
        "visible",
        "visible_in_vehicle",
        "offscreen",
        "in_structure",
        "in_vehicle",
        "unknown",
        "exited"
    ]
    vehicle_id: Optional[int]  # If state == "in_vehicle" or "visible_in_vehicle"
    structure_id: Optional[str]  # If state == "in_structure"
    last_track_id: int
    last_seen_timestamp: float
```

**Vehicle agent:**
```python
class VehicleAgent:
    agent_id: int
    state: Literal[
        "visible_parked",
        "visible_moving",
        "offscreen",
        "in_structure",
        "unknown",
        "exited"
    ]
    structure_id: Optional[str]  # If state == "in_structure"
    occupants: List[OccupantAssignment]
    last_seen_timestamp: float

class OccupantAssignment:
    agent_id: int
    entry_timestamp: float
    entry_track_id: int
    sporadic_detections: List[Detection]  # Face detections through windows
    validated: bool
```

### Entry Detection Algorithm
```python
def detect_vehicle_entry(person_track, vehicle_track):
    """
    Correlate person disappearance with vehicle movement
    """
    # Temporal correlation
    time_delta = vehicle_track.start_time - person_track.end_time
    if time_delta < 0 or time_delta > 5.0:  # seconds
        return False

    # Spatial correlation
    distance = euclidean_distance(
        person_track.end_position,
        vehicle_track.start_position
    )
    if distance > VEHICLE_PROXIMITY_THRESHOLD:  # e.g., 5 meters
        return False

    # Update states
    person_agent.state = "in_vehicle"
    person_agent.vehicle_id = vehicle_track.agent_id
    vehicle_agent.occupants.append(OccupantAssignment(
        agent_id=person_agent.id,
        entry_timestamp=person_track.end_time,
        entry_track_id=person_track.id
    ))

    return True
```

### Sporadic Interior Detection Handling
```python
def process_interior_detection(vehicle_track, detection):
    """
    Handle person detection inside vehicle (through windows)

    Don't require frame-to-frame continuity
    """
    if detection.class_name != "person":
        return

    # Attempt face match against vehicle occupants
    for occupant in vehicle_agent.occupants:
        face_similarity = face_recognition.compare(
            detection.crop,
            occupant.agent.face_library
        )
        if face_similarity > 0.75:
            # Strengthen assignment
            occupant.sporadic_detections.append(detection)
            occupant.validated = True  # Face match confirms
            return

    # Unknown face in vehicle - possible additional occupant
    # Log but don't force assignment
```

### Exit Detection Algorithm
```python
def detect_vehicle_exit(vehicle_track, person_track):
    """
    Correlate vehicle parking with person appearance
    """
    # Check vehicle is parked
    if vehicle_agent.state != "parked":
        return None

    # Temporal correlation
    time_delta = person_track.start_time - vehicle_track.end_time
    if time_delta < 0 or time_delta > 10.0:
        return None

    # Spatial correlation
    distance = euclidean_distance(
        vehicle_track.end_position,
        person_track.start_position
    )
    if distance > VEHICLE_PROXIMITY_THRESHOLD:
        return None

    # Find occupant candidates
    candidates = vehicle_agent.occupants

    # Evidence evaluation
    if len(candidates) == 1:
        return AUTO_MERGE(candidates[0].agent_id, person_track)

    # Multiple candidates - use face match
    for candidate in candidates:
        if person_track.has_face:
            similarity = face_recognition.compare(
                person_track.best_face_crop,
                candidate.agent.face_library
            )
            if similarity > 0.75:
                return AUTO_MERGE(candidate.agent_id, person_track)

    # No face match - check process of elimination
    unaccounted_candidates = [
        c for c in candidates
        if not c.agent.has_conflicting_track(person_track.start_time)
    ]

    if len(unaccounted_candidates) == 1:
        return MERGE_WITH_UNCERTAINTY(unaccounted_candidates[0], person_track)

    # Multiple unaccounted - uncertain
    return MULTI_ASSIGNMENT(unaccounted_candidates, person_track)
```

### Process of Elimination
```python
def rule_out_vehicle_occupant(agent_id, timestamp):
    """
    Agent seen elsewhere - can't be in vehicle

    Cascades through all vehicles with this occupant
    """
    for vehicle in active_vehicles:
        for occupant in vehicle.occupants:
            if occupant.agent_id == agent_id:
                # Remove from occupant list
                vehicle.occupants.remove(occupant)

                # Cascade: if only one occupant remains, increase confidence
                if len(vehicle.occupants) == 1:
                    vehicle.occupants[0].validated = True
```

### Portal-Based Merging (Entry and Exit)

**Background:** Portals are configured entry/exit points where an agent can become occluded (building entrance, garage, dense vegetation). Normal grid-based timing doesn't apply because the agent's whereabouts are unknown while in the portal.

**Key insight:** Portal crossings are **identity-preserving boundaries**. If Track A disappears into a portal and Track B emerges from the same portal, they represent the same agent (subject to constraints).

#### Portal Entry: Track Disappears Into Portal

```python
def evaluate_portal_entry(track: Track, portal: Portal) -> PortalEntryResult:
    """
    Track ends at a portal location. Update agent state to reflect entry into occluded space.
    """
    agent = get_agent(track.agent_id)

    # Spatial check: Is track exit near portal location?
    dist = distance(track.exit_cell, portal.location)
    if dist > PORTAL_ENTRY_THRESHOLD:
        return REJECTED, "too_far_from_portal"

    # Update agent state
    if agent.agent_type == "person":
        agent.state = f"in_structure({portal.structure_id})"
        agent.structure_entry_time = track.end_time
        # Timeline: "Person entered building at East Door"

    elif agent.agent_type == "vehicle":
        agent.state = f"in_structure({portal.structure_id})"
        agent.structure_entry_time = track.end_time
        # Timeline: "Vehicle entered garage"

    return PORTAL_ENTRY_ACCEPTED, portal.structure_id
```

#### Portal Exit: Track Emerges From Portal

```python
def evaluate_portal_exit_merge(
    new_track: Track,
    portal: Portal,
    portal_entry_agents: Dict[str, float]  # {agent_id: entry_time}
) -> PortalExitResult:
    """
    New track starts near portal exit point. Could it be an agent that entered the portal earlier?

    Plausible candidates: Any agent with state=in_structure(portal.structure_id)
    """
    # Spatial check: Is new track entry near portal location?
    dist = distance(new_track.entry_cell, portal.location)
    if dist > PORTAL_EXIT_THRESHOLD:
        return REJECTED, "too_far_from_portal"

    # Find candidates: agents currently in this structure
    candidates = [
        agent_id
        for agent_id, entry_time in portal_entry_agents.items()
        if get_agent(agent_id).state == f"in_structure({portal.structure_id})"
    ]

    if len(candidates) == 0:
        # No one known to be in structure - new agent
        return NEW_AGENT, "structure_was_empty"

    if len(candidates) == 1:
        # Only one possibility - deterministic
        candidate_id = candidates[0]
        candidate_agent = get_agent(candidate_id)

        # Time window check: Is dwell time plausible?
        dwell_time = new_track.start_time - candidate_agent.structure_entry_time
        if dwell_time < PORTAL_MIN_DWELL or dwell_time > PORTAL_MAX_DWELL:
            # Time doesn't fit, but be lenient (user might have walked around inside)
            return AUTO_MERGE, f"portal_exit_single_candidate_{candidate_id}"

        return AUTO_MERGE, f"portal_exit_deterministic_{candidate_id}"

    # Multiple candidates: use secondary evidence
    return evaluate_portal_exit_ambiguous(new_track, candidates, portal)

def evaluate_portal_exit_ambiguous(
    new_track: Track,
    candidates: List[str],
    portal: Portal
) -> PortalExitResult:
    """
    Multiple people in same structure. Which one is exiting?
    Apply process of elimination.
    """
    candidate_scores = {}

    for candidate_id in candidates:
        candidate = get_agent(candidate_id)
        score = 0.0

        # EVIDENCE 1: Face match
        face_sim = compute_face_similarity(new_track.face, candidate.face_library)
        if face_sim > 0.7:
            score += 40

        # EVIDENCE 2: Re-ID match
        reid_sim = compute_reid_similarity(new_track.reid, candidate.reid_embedding)
        if reid_sim > 0.7:
            score += 30

        # EVIDENCE 3: Dwell time plausibility
        dwell_time = new_track.start_time - candidate.structure_entry_time
        if PORTAL_MIN_DWELL <= dwell_time <= PORTAL_MAX_DWELL:
            score += 20

        # EVIDENCE 4: Interior detections during dwell
        if candidate_id in portal.interior_detections:
            score += 15

        # EVIDENCE 5: Clothing consistency
        if clothing_similarity(new_track, candidate) > 0.8:
            score += 10

        candidate_scores[candidate_id] = score

    # Sort by score
    best_candidate, best_score = max(candidate_scores.items(), key=lambda x: x[1])

    if best_score < 20:
        # Even best candidate has low confidence
        return AMBIGUOUS, f"portal_exit_low_confidence_{candidates}"

    # Check if second-best is close to best
    scores_sorted = sorted(candidate_scores.values(), reverse=True)
    if len(scores_sorted) > 1 and scores_sorted[0] - scores_sorted[1] < 10:
        # Close call between candidates
        return AMBIGUOUS, f"portal_exit_ambiguous_{candidates}"

    # Best candidate significantly better than others
    return AUTO_MERGE, f"portal_exit_best_match_{best_candidate}"
```

#### Portal Re-Entry: Agent Exits Portal and Re-Enters

**Scenario:** Person enters building at time T1, exits at T2, then re-enters at T3.

```python
def handle_multiple_portal_crossings(agent: Agent, portal: Portal):
    """
    Agent crosses same portal multiple times.
    Track each crossing separately in agent.portal_history.
    """
    # Portal history tracks each crossing
    # Entry: timestamp, location
    # Exit: timestamp, location
    #
    # Example:
    # portal_history = [
    #   {type: "entry", time: 100, location: portal.id},
    #   {type: "exit", time: 150, location: portal.id},
    #   {type: "entry", time: 160, location: portal.id},
    #   {type: "exit", time: 200, location: portal.id},
    # ]
    #
    # This maintains full picture for timeline reconstruction

    # Agent can re-enter same portal
    # Each time, determine which exit corresponds to which entry
    # Use time proximity + location matching
```

### Person-in-Vehicle Merge Exemption
```python
def find_merge_candidates(person_agent):
    """
    Find merge candidates for person tracks

    IMPORTANT: Skip person if in vehicle (either state)
    """
    if person_agent.state in ("in_vehicle", "visible_in_vehicle"):
        # Person movement is vehicle movement
        # Don't look for person→person merges across cameras
        # Sporadic visible_in_vehicle detections are for face matching only
        # Vehicle track is the proxy for movement
        return []

    # Normal merge candidate logic for visible/offscreen/in_structure/unknown
    return find_cross_camera_candidates(person_agent)
```

### Vehicle Portal Transitions
```python
def detect_vehicle_portal_transition(vehicle_track, portal):
    """
    Vehicle can enter/exit garage (portal: garage door)

    State transitions:
    - visible_moving → in_structure(Garage) (entered garage)
    - in_structure(Garage) → visible_moving (exited garage)
    - visible_moving → exited (left property via exit portal)
    """
    if portal.type == "garage_door":
        if vehicle_track.direction == "entering":
            vehicle_agent.state = "in_structure"
            vehicle_agent.structure_id = portal.structure_id
        elif vehicle_track.direction == "exiting":
            vehicle_agent.state = "visible_moving"
            vehicle_agent.structure_id = None

    elif portal.type == "property_exit":
        vehicle_agent.state = "exited"

        # Occupants also exited property (still in vehicle)
        for occupant in vehicle_agent.occupants:
            person_agent = get_agent(occupant.agent_id)
            person_agent.state = "exited"  # Person left property via vehicle
            # Timeline notes "left property in Vehicle V"
```

## Related Documents
- **L12_TrackAgentRelationship:** How tracks map to agents (foundation for merge model)
- **L12_ConcurrentTracking:** Multi-agent scenarios on single camera (group tracks excluded from merging)
- **L12_ProcessOfElimination:** System-wide pattern for resolving identity uncertainty (core concept for merge evaluation)
- **L12_VehicleOccupancy:** Vehicle-specific merge logic
- **L3_EvidenceEngine:** Fact-gathering for merge candidates
- **L3_TrackMerging:** Non-technical approach
- **L2_Strategy:** High-level evidence hierarchy and grid-based strategy
- **L4_Complexity:** Where merging gets hard
