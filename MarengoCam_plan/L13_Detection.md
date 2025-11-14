# Level 13 - Detection and Tracking Technical Specs

## Purpose
Technical implementation of detection, tracking, and chunk linking.

## Chunked Processing Architecture

### Chunk Structure
```python
class TrackChunk:
    """
    Single YOLO tracking session (12s video chunk, one camera)
    """
    chunk_id: int          # Sequential chunk number (0, 1, 2, ...)
    camera_id: int
    start_time: float      # Absolute timestamp (e.g., T=0, T=10, T=20...)
    duration: float = 12.0 # seconds
    overlap_start: float   # First 2s overlap with previous chunk
    overlap_end: float     # Last 2s overlap with next chunk

    # YOLO tracking results
    local_tracks: Dict[int, LocalTrack]  # YOLO local ID → track data
    frames: List[Frame]    # Raw frame data (12s @ sampling rate)

class LocalTrack:
    """
    Single agent's track within one 12s chunk
    """
    local_id: int          # YOLO tracker ID (only valid within this chunk)
    global_agent_id: Optional[int]  # Mapped to global agent (if known)

    camera_id: int
    chunk_id: int

    # Temporal extent
    start_time: float      # Relative to chunk start (0-12s)
    end_time: float
    start_frame: int
    end_frame: int

    # Spatial trajectory
    detections: List[Detection]  # Frame-by-frame bboxes, positions

    # Linking: temporal continuity (same camera, sequential chunks)
    previous_chunk_track: Optional[LocalTrack]  # Same agent in previous chunk
    next_chunk_track: Optional[LocalTrack]      # Same agent in next chunk

    # Linking: spatial continuity (cross-camera merges)
    cross_camera_merges: List[LocalTrack]  # Same agent on other cameras

    # Track quality metrics
    average_confidence: float
    movement_score: float  # Sustained motion metric
    duration: float        # Total track duration (seconds)

class Detection:
    """
    Single frame detection within a track
    """
    frame_number: int
    timestamp: float
    bbox: BoundingBox     # x, y, w, h (normalized 0-1)
    confidence: float
    position: Tuple[float, float]  # Center position (normalized)
    grid_cell: int        # 6×6 grid cell index

    # Dual crop strategy
    body_crop: bytes      # Full body JPEG crop (for LLM panels)
    face_crop: Optional[bytes]  # Face-only crop (for face recognition)
    face_bbox: Optional[BoundingBox]  # Face location within body crop

    # Quality metrics
    blur_score: float     # Lower is sharper (Laplacian variance)
    brightness: float     # Mean luminance (0-255)
    face_visible: bool    # Face detector found face
    face_quality: Optional[float]  # Face-specific quality score
```

### Chunk Overlap Linking

**Match local IDs across chunk boundary:**
```python
def link_chunk_overlap(chunk_n: TrackChunk, chunk_n_plus_1: TrackChunk):
    """
    Match YOLO local IDs across 2-second overlap zone

    Overlap window: last 2s of chunk_n == first 2s of chunk_n_plus_1
    """
    # Get tracks active in overlap zone
    tracks_n_end = [
        t for t in chunk_n.local_tracks.values()
        if t.end_time >= 10.0  # Active in last 2s (T=10-12s)
    ]

    tracks_n1_start = [
        t for t in chunk_n_plus_1.local_tracks.values()
        if t.start_time <= 2.0  # Active in first 2s (T=0-2s)
    ]

    # Match tracks
    for track_n in tracks_n_end:
        for track_n1 in tracks_n1_start:
            if match_tracks_in_overlap(track_n, track_n1):
                # Link tracks
                track_n.next_chunk_track = track_n1
                track_n1.previous_chunk_track = track_n

                # Propagate global agent ID
                if track_n.global_agent_id is not None:
                    track_n1.global_agent_id = track_n.global_agent_id

def match_tracks_in_overlap(track_n: LocalTrack, track_n1: LocalTrack) -> bool:
    """
    Determine if two tracks from adjacent chunks are the same agent

    Uses:
    1. Spatial continuity (position at chunk boundary)
    2. Appearance similarity (if multiple candidates)
    3. Agent type match
    """
    # Check agent type
    if track_n.agent_type != track_n1.agent_type:
        return False

    # Spatial continuity: position at T=12s (chunk_n) vs T=10s (chunk_n1 absolute time)
    # Note: chunk_n1 T=0 is chunk_n T=10
    position_n_end = track_n.detections[-1].position
    position_n1_start = track_n1.detections[0].position

    distance = euclidean_distance(position_n_end, position_n1_start)

    # Deterministic if single agent
    if count_candidates_in_overlap(track_n, track_n1) == 1:
        # Only one agent of this type in overlap → must be same
        return distance < OVERLAP_POSITION_THRESHOLD  # e.g., 0.1 (10% of frame)

    # Multiple candidates: check appearance similarity
    if distance < OVERLAP_POSITION_THRESHOLD:
        appearance_score = compare_appearance(
            track_n.detections[-1].crop,
            track_n1.detections[0].crop
        )
        return appearance_score > APPEARANCE_THRESHOLD  # e.g., 0.7

    return False
```

### Global Agent ID Assignment

**Map local track IDs to global agents:**
```python
class GlobalAgent:
    """
    Agent across all cameras and all chunks
    """
    global_id: int
    type: Literal["person", "vehicle", "animal"]

    # All track chunks belonging to this agent
    track_chunks: List[LocalTrack]  # Chronologically ordered

    # Current state
    state: Literal["visible", "in_vehicle", "in_structure", "unknown"]
    last_seen_timestamp: float
    last_seen_camera: int

    # Face library (for people)
    face_embeddings: List[np.ndarray]

def assign_global_agent_id(local_track: LocalTrack) -> int:
    """
    Assign or reuse global agent ID for a local track.

    Note: local_track is created from a promoted CandidateTrack.
    See L13_TrackStateManager.promote_candidate_to_agent() for candidate lifecycle.

    Cases:
    1. Chunk continuation (has previous_chunk_track) → inherit global ID
    2. Cross-camera merge (grid overlap, face match, LLM) → merge to existing agent
    3. New agent → create new global ID
    """
    # Case 1: Continuation from previous chunk
    if local_track.previous_chunk_track is not None:
        return local_track.previous_chunk_track.global_agent_id

    # Case 2: Cross-camera merge candidate?
    merge_candidates = find_cross_camera_merge_candidates(local_track)

    if len(merge_candidates) == 1:
        # Definitive merge (grid overlap single agent, face match, etc.)
        return merge_candidates[0].global_agent_id
    elif len(merge_candidates) > 1:
        # Multi-assignment (uncertain)
        local_track.agent_assignments = [
            AgentAssignment(agent_id=c.global_agent_id, evidence=["candidate"])
            for c in merge_candidates
        ]
        return None  # Unresolved

    # Case 3: New agent
    new_agent = create_global_agent(local_track.agent_type)
    return new_agent.global_id
```

## YOLO Integration

### Inference
```python
def process_chunk(video_chunk: bytes, camera_id: int, start_time: float) -> TrackChunk:
    """
    Run YOLO11 tracking on 12-second video chunk
    """
    from ultralytics import YOLO

    model = YOLO('yolo11n.pt')  # Nano model for speed

    # Run tracking (YOLO maintains IDs across frames)
    results = model.track(
        source=video_chunk,
        stream=True,
        tracker='botsort.yaml',  # Or 'bytetrack.yaml'
        classes=[0, 2, 15, 16],  # person, car, cat, dog
        conf=0.5,
        iou=0.5
    )

    # Parse results into TrackChunk structure
    chunk = TrackChunk(
        camera_id=camera_id,
        start_time=start_time,
        duration=12.0
    )

    for frame_idx, result in enumerate(results):
        for box in result.boxes:
            track_id = int(box.id) if box.id is not None else None
            if track_id is None:
                continue  # Skip untracked detections

            # Get or create LocalTrack
            if track_id not in chunk.local_tracks:
                chunk.local_tracks[track_id] = LocalTrack(
                    local_id=track_id,
                    camera_id=camera_id,
                    chunk_id=chunk.chunk_id,
                    start_time=frame_idx / FPS,
                    detections=[]
                )

            # Add detection to track
            detection = Detection(
                frame_number=frame_idx,
                timestamp=start_time + frame_idx / FPS,
                bbox=box.xyxyn[0].tolist(),
                confidence=float(box.conf),
                position=(float(box.xywhn[0][0]), float(box.xywhn[0][1])),
                **extract_dual_crops(result.orig_img, box.xyxy[0], box.cls)
            )

            chunk.local_tracks[track_id].detections.append(detection)

    # Update end times for all tracks
    for track in chunk.local_tracks.values():
        track.end_time = track.detections[-1].frame_number / FPS
        track.duration = track.end_time - track.start_time

    return chunk

def extract_dual_crops(frame: np.ndarray, bbox: np.ndarray, class_id: int) -> dict:
    """
    Extract both body crop and face crop from detection

    Returns dict with:
    - body_crop: bytes
    - face_crop: Optional[bytes]
    - face_bbox: Optional[BoundingBox]
    - blur_score: float
    - brightness: float
    - face_visible: bool
    - face_quality: Optional[float]
    """
    x1, y1, x2, y2 = bbox.astype(int)

    # Extract body crop (full detection bbox)
    body_region = frame[y1:y2, x1:x2]
    body_crop = cv2.imencode('.jpg', body_region, [cv2.IMWRITE_JPEG_QUALITY, 90])[1].tobytes()

    # Compute quality metrics for body crop
    blur_score = compute_blur(body_region)
    brightness = compute_brightness(body_region)

    # Face detection (for person class only)
    face_crop = None
    face_bbox = None
    face_visible = False
    face_quality = None

    if class_id == 0:  # person class
        # Run face detector on body crop
        face_results = detect_faces(body_region)

        if len(face_results) > 0:
            # Take largest face
            face = max(face_results, key=lambda f: f.area())
            fx1, fy1, fx2, fy2 = face.bbox

            # Extract face crop
            face_region = body_region[fy1:fy2, fx1:fx2]
            face_crop = cv2.imencode('.jpg', face_region, [cv2.IMWRITE_JPEG_QUALITY, 95])[1].tobytes()

            # Face bbox relative to body crop
            face_bbox = BoundingBox(
                x=fx1 / body_region.shape[1],
                y=fy1 / body_region.shape[0],
                w=(fx2 - fx1) / body_region.shape[1],
                h=(fy2 - fy1) / body_region.shape[0]
            )

            face_visible = True
            face_quality = compute_face_quality(face_region)

    return {
        "body_crop": body_crop,
        "face_crop": face_crop,
        "face_bbox": face_bbox,
        "blur_score": blur_score,
        "brightness": brightness,
        "face_visible": face_visible,
        "face_quality": face_quality
    }

def compute_blur(image: np.ndarray) -> float:
    """
    Compute blur score using Laplacian variance

    Lower score = blurrier image
    """
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    laplacian_var = cv2.Laplacian(gray, cv2.CV_64F).var()
    return float(laplacian_var)

def compute_brightness(image: np.ndarray) -> float:
    """
    Compute mean brightness (0-255)
    """
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    return float(gray.mean())

def compute_face_quality(face_image: np.ndarray) -> float:
    """
    Compute face-specific quality score

    Factors:
    - Sharpness
    - Size (larger is better)
    - Frontality (facing camera)
    """
    # Sharpness
    blur = compute_blur(face_image)
    sharpness_score = min(1.0, blur / 500.0)  # Normalize

    # Size (prefer larger faces)
    size_score = min(1.0, (face_image.shape[0] * face_image.shape[1]) / 10000.0)

    # TODO: Frontality detection (face landmarks, head pose estimation)
    frontality_score = 0.8  # Placeholder

    # Weighted combination
    quality = 0.5 * sharpness_score + 0.3 * size_score + 0.2 * frontality_score
    return quality

def detect_faces(image: np.ndarray) -> List[Face]:
    """
    Detect faces in image using OpenCV/dlib/MTCNN

    Returns list of Face objects with bbox and landmarks
    """
    # TODO: Implement face detection (OpenCV Haar, dlib HOG, MTCNN, etc.)
    # For now, placeholder
    return []
```

## Chunk Pipeline

**Multi-camera parallel processing:**
```python
def chunk_processor_pipeline(cameras: List[int], start_time: float):
    """
    Process 12s chunks across multiple cameras in parallel

    Pipeline stages:
    1. Acquire video chunks (Blue Iris API)
    2. YOLO inference (GPU-bound, batched)
    3. Overlap linking (CPU-bound)
    4. Cross-camera merging (CPU-bound)
    5. Global agent ID assignment
    """
    # Stage 1: Acquire chunks (parallel)
    chunk_futures = [
        acquire_video_chunk_async(cam, start_time, duration=12.0)
        for cam in cameras
    ]

    # Stage 2: YOLO inference (batched across cameras for GPU efficiency)
    track_chunks = batch_yolo_inference(chunk_futures)

    # Stage 3: Overlap linking (for each camera, link to previous chunk)
    for cam in cameras:
        prev_chunk = get_previous_chunk(cam, start_time)
        curr_chunk = track_chunks[cam]
        if prev_chunk is not None:
            link_chunk_overlap(prev_chunk, curr_chunk)

    # Stage 4: Cross-camera merging (grid-based, face recognition)
    for cam_a in cameras:
        for cam_b in cameras:
            if cam_a >= cam_b:
                continue  # Avoid duplicate pairs
            merge_tracks_cross_camera(track_chunks[cam_a], track_chunks[cam_b])

    # Stage 5: Global agent ID assignment
    for chunk in track_chunks.values():
        for track in chunk.local_tracks.values():
            if track.global_agent_id is None:
                track.global_agent_id = assign_global_agent_id(track)

    # Persist to database
    save_chunks_to_database(track_chunks)

    # Schedule next cycle (T+10s, overlapping)
    schedule_next_cycle(start_time + 10.0)
```

## Related Documents
- **L3_Detection:** Non-technical approach to chunking and tracking
- **L12_Merging:** Cross-camera merge technical specs
- **L11_Data:** Database schema for storing chunks and tracks
