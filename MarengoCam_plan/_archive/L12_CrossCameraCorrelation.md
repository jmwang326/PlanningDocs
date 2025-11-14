# Level 12 - Cross-Camera Correlation Architecture

## 1. Purpose
This document provides the high-level technical architecture for the cross-camera correlation and identity persistence strategy. It defines the data structures, component responsibilities, and external API contracts required to build a coherent timeline of agent activity across multiple camera views.

This architecture is implemented by the tactical components defined in the L13 documents, including the Track Merging Engine, Evidence Engine, and Track State Manager.

---

## 2. Agent State & Data Structures

### 2.1. Core Data Models
The following data structures define the primary entities managed by the correlation services.

```python
# High-level structure for a person agent
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
    vehicle_id: Optional[int]
    structure_id: Optional[str]
    last_track_id: int
    last_seen_timestamp: float

# High-level structure for a vehicle agent
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
    structure_id: Optional[str]
    occupants: List[OccupantAssignment]
    last_seen_timestamp: float

# Association of a person to a vehicle
class OccupantAssignment:
    agent_id: int
    entry_timestamp: float
    entry_track_id: int
    sporadic_detections: List[Detection]
    validated: bool

# Structure for a track with multi-agent uncertainty
class Track:
    agent_assignments: List[AgentAssignment]

class AgentAssignment:
    agent_id: int
    evidence: List[str]
    validated: bool
```

### 2.2. Detection Data Model
Each detection must store dual crops to support different services.

```python
class Detection:
    bbox: BoundingBox           # Full detection bounding box
    body_crop: bytes            # Full body JPEG crop (for LLM/human review)
    face_crop: Optional[bytes]  # Face-only crop (for face recognition)
    face_bbox: Optional[BoundingBox]
    # Plus quality metrics like blur, brightness, etc.
```

---

## 3. Component Interfaces & Responsibilities

### 3.1. Merge Candidate Finding
- **Responsibility:** Periodically identify potential track-to-track merges across cameras.
- **Interface:** `find_merge_candidates() -> Generator[(Track, Track)]`
- **High-Level Logic:** Finds recent tracks and yields pairs that are plausible in time and space, and are connected by a known portal or overlap.

### 3.2. Evidence Scoring
- **Responsibility:** Evaluate a merge candidate pair and return a decision.
- **Interface:** `evaluate_merge_candidate(track1: Track, track2: Track) -> Tuple[Decision, str]`
- **High-Level Logic:** Uses a hierarchy of evidence (face match, grid overlap, portal stats) to classify a candidate as `AUTO_MERGE`, `REJECT`, `REVIEW_LLM`, etc.

### 3.3. Vehicle Entry/Exit Detection
- **Responsibility:** Detect when a person enters or exits a vehicle.
- **Interfaces:**
    - `detect_vehicle_entry(person_track: Track, vehicle_track: Track) -> bool`
    - `detect_vehicle_exit(vehicle_track: Track, person_track: Track) -> Optional[MergeDecision]`
- **High-Level Logic:** Correlates person and vehicle tracks based on spatial and temporal proximity.

### 3.4. Panel Assembly
- **Responsibility:** Create visual panels for LLM or human review.
- **Interface:** `assemble_comparison_panel(track_a: Track, track_b: Track) -> bytes`
- **High-Level Logic:** Selects the best K frames from each track and assembles them into a grid image.

---

## 4. External Service Contracts

### 4.1. Face Recognition (CodeProject.AI)
- **Service:** Provides face matching and a persistent face library.
- **Key Endpoints:**
    - `POST /v1/vision/face/recognize`
    - `POST /v1/vision/face/register`
    - `GET /v1/vision/face/list`
    - `DELETE /v1/vision/face/{userid}`

### 4.2. LLM Adjudication (Gemini)
- **Service:** Provides a judgment on ambiguous merge candidates.
- **Interaction:** A JSON-based prompt is sent containing image panels and context, expecting a structured JSON response (`"answer": "YES|NO|CANNOT"`).

---

## 5. Identity & Re-ID Library Management

### 5.1. Face & Vehicle Libraries
- **Goal:** Build and maintain libraries of face crops and vehicle appearances to improve recognition over time.
- **Strategy:** Populate libraries from high-confidence, validated merges, ensuring diversity across cameras, lighting conditions, and orientations.
- **Data Structures:**
```python
class FaceLibrary:
    agent_id: int
    user_id: str
    faces: List[RegisteredFace]

class RegisteredFace:
    face_crop: bytes
    camera_id: int
    lighting_condition: str
    timestamp: float
    quality_score: float

class VehicleReIDLibrary:
    agent_id: int
    vehicle_id: str
    appearances: List[VehicleAppearance]

class VehicleAppearance:
    body_crop: bytes
    camera_id: int
    orientation: str
    timestamp: float
    quality_score: float
```

### 5.2. Manual Identity Management
- **Requirement:** A robust UI for manual identity assignment is required for the human-in-the-loop workflow.
- **Features:** Must include a central identity registry, a searchable "Identity Picker" with fuzzy matching to prevent duplicates, and a clear workflow for creating new identities.

---

## 6. Inter-Camera Grid
- **Purpose:** To learn and store the spatial and temporal relationships between camera views.
- **Data Structure:**
```python
class InterCameraGrid:
    camera_id: int
    transitions: Dict[int, Dict[int, Dict[int, TransitionStats]]]

class TransitionStats:
    typical_time: float
    variance: float
    sample_count: int
    last_updated: float
```
- **Learning:** The grid is bootstrapped and refined based on high-confidence merges validated by face recognition or human review.

## Related Documents
- **L3_TrackMerging:** Tactical implementation of the merging algorithms.
- **L3_EvidenceEngine:** Tactical implementation of evidence scoring.
- **L3_PanelAssemblyService:** Tactical implementation of panel creation.
- **L3_TrackStateManager:** Manages agent state transitions.
