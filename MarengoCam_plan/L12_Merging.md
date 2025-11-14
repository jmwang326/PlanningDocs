# Level 12 - Merging Technical Specs

## Purpose
Technical implementation of cross-camera merge logic.

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

### Evidence Scoring (with Grid)
**Hierarchy with grid overlap detection:**
```python
def evaluate_merge_candidate(track1, track2):
    """
    Evidence hierarchy incorporating grid overlap
    """
    # 1. Face recognition (definitive)
    if face_match(track1, track2) > 0.75:
        return AUTO_MERGE, "face_match"

    # 2. Grid overlap + single agent (auto-merge)
    grid_stats = get_grid_transition_stats(
        track1.camera, track1.end_cell,
        track2.camera, track2.start_cell
    )

    if grid_stats and grid_stats.is_overlap():  # typical_time < 1.0
        # Check type compatibility
        if track1.agent_type != track2.agent_type:
            return REJECT, "type_mismatch"

        # Check person-in-vehicle exemption
        if track1.agent_type == "person" and track1.agent.state == "in_vehicle":
            return REJECT, "person_in_vehicle_exemption"

        # Count agents of same type in overlap zones
        agents_zone1 = count_agents_in_zone(track1.camera, track1.end_cell, track1.end_time)
        agents_zone2 = count_agents_in_zone(track2.camera, track2.start_cell, track2.start_time)

        if agents_zone1 == 1 and agents_zone2 == 1:
            # Single agent in overlap → must be same
            return AUTO_MERGE, "grid_overlap_single"
        else:
            # Multi-agent overlap → constrained multi-assignment
            return MULTI_ASSIGN_OVERLAP, "grid_overlap_multi"

    # 3. Grid portal + time/space gating
    if grid_stats and grid_stats.is_portal():  # 1.0 <= typical_time <= 15.0
        time_delta = track2.start_time - track1.end_time

        # Check time fits typical ± margin
        margin = max(2.0, grid_stats.variance * 2)  # At least 2s margin
        if abs(time_delta - grid_stats.typical_time) > margin:
            return REJECT, "time_mismatch_portal"

        # Portal candidate → continue to LLM/human
        return REVIEW_LLM, "portal_candidate"

    # 4. No grid data
    if not grid_stats:
        # Fall back to manual portal config
        if manual_portal_exists(track1.camera, track2.camera):
            return REVIEW_LLM, "manual_portal"
        else:
            return REJECT, "no_adjacency"

    # 5. Default: reject
    return REJECT, "no_evidence"
```

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

## Inter-Camera Grid (6×6)

### Data Structure
```python
class InterCameraGrid:
    """
    6×6 grid per camera for learning spatial adjacency
    """
    camera_id: int
    grid_size: Tuple[int, int] = (6, 6)  # 36 cells

    # Sparse storage: only cell pairs with observed transitions
    transitions: Dict[int, Dict[int, Dict[int, TransitionStats]]]
    # transitions[cell_id][other_camera_id][other_cell_id] = TransitionStats

class TransitionStats:
    """
    Learned statistics for cell pair transitions
    """
    typical_time: float  # Median transition time (seconds)
    variance: float      # Variance in transition times
    sample_count: int    # Number of validated transitions
    last_updated: float  # Timestamp

    def is_overlap(self) -> bool:
        """Overlap if typical time < 1s (simultaneous detection)"""
        return self.typical_time < 1.0

    def is_portal(self) -> bool:
        """Portal if 1-15s transition time"""
        return 1.0 <= self.typical_time <= 15.0

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
def learn_from_validated_merge(track_a, track_b):
    """
    User/LLM validated: Track A and Track B are same agent
    Update grid transition statistics
    """
    cell_a = position_to_grid_cell(track_a.end_position)
    cell_b = position_to_grid_cell(track_b.start_position)

    time_delta = track_b.start_time - track_a.end_time

    # Get or create transition stats
    if cell_a not in grid[track_a.camera].transitions:
        grid[track_a.camera].transitions[cell_a] = {}
    if track_b.camera not in grid[track_a.camera].transitions[cell_a]:
        grid[track_a.camera].transitions[cell_a][track_b.camera] = {}
    if cell_b not in grid[track_a.camera].transitions[cell_a][track_b.camera]:
        grid[track_a.camera].transitions[cell_a][track_b.camera][cell_b] = TransitionStats()

    # Update statistics
    stats = grid[track_a.camera].transitions[cell_a][track_b.camera][cell_b]
    stats.update(time_delta)

    # Grid now knows:
    # - If time_delta ≈ 0: cameras overlap at these cells
    # - If time_delta 1-15s: portal connection
    # - If time_delta > 15s: distant cameras, rare transition
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
- **Source:** AGENT_TRACKING_AND_MERGE.md (deprecated)
- **L3_Merging:** Non-technical approach
- **L4_Complexity:** Where this gets hard
