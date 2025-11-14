# Level 12 - System Architecture

## Purpose
This document outlines the high-level technical components (core processes) of the MarengoCam system. These components are the technical implementation of the strategies defined in `L2_Strategy.md`. Each component represents a major functional block of the system.

## Core Real-Time Components

### 1. Frame Acquisition (Blue Iris Integration)
- **Function:** Fetches JPEG snapshots from Blue Iris HTTP endpoints at configured frame rate (10 FPS production, 2 FPS DEV_SLOW).
- **Responsibilities:** Polls Blue Iris `/image/{short}?q=60` endpoint for sub and main profiles, handles connection errors, maintains frame buffers for pre-roll/post-roll.
- **Corresponds to L3 Tactic:** `L3_Acquisition.md`

### 2. Detection & Intra-Camera Tracking
- **Function:** Processes video chunks using the object detection and tracking model (YOLOv11).
- **Responsibilities:** Identifies objects (people, vehicles) in frames and links them across frames within a single camera's view to create initial "tracklets." Filters out initial noise based on persistence and movement.
- **Corresponds to L3 Tactic:** `L3_Detection.md`

### 3. Track State Manager
- **Function:** The central state machine for the entire system.
- **Responsibilities:** Manages the lifecycle of all tracks and agents. Tracks states include `new`, `active`, `lost`, `uncertain_merge`, `merged`, and `archived`. This component orchestrates the flow of data between the other components.

### 4. Evidence Engine
- **Function:** Gathers all possible evidence for potential track merges.
- **Responsibilities:** For any two tracks, it calculates face recognition similarity, spatial/temporal proximity, visual similarity (e.g., clothing color), and checks for portal crossings. It provides this evidence package to the Merging Engine.

### 5. Track Merging Engine
- **Function:** Decides whether to merge two or more tracklets into a single "agent" timeline.
- **Responsibilities:** Applies the evidence hierarchy defined in `L2_Strategy.md`. Executes merges based on definitive evidence (face match) or weighted evidence, and flags merges that are uncertain.
- **Corresponds to L3 Tactic:** `L3_Merging.md`

### 6. Timeline Reconstruction
- **Function:** Converts the raw data of merged agent tracks into a human-readable narrative.
- **Responsibilities:** Generates descriptive event logs (e.g., "Person A entered via East Gate at 10:32 AM") from the agent's path through camera zones and portals.

### 7. Panel Assembly Service
- **Function:** A specialized, stateless image processing service.
- **Responsibilities:** Creates standardized visual artifacts (e.g., 2x6 comparison panels) for review. It handles all image fetching, cropping, resizing, upscaling, and composition, returning only a file path to the final image.

### 8. Presentation Layer (GUI)
- **Function:** The user-facing interface for viewing and interacting with the system's output.
- **Responsibilities:** Displays the reconstructed agent timelines, allows users to review video clips, and provides the interface for the "Human-in-the-Loop" review process by presenting tasks from the review queue.

## Supporting Systems

### 9. Configuration System
- **Function:** Provides configuration data to all other components.
- **Responsibilities:** Manages camera settings, portal definitions, detection thresholds, and evidence tuning parameters. This is the system's central control panel.

### 10. Learning & Training Subsystem
- **Function:** Offline processes responsible for improving the system's accuracy over time.
- **Sub-components:**
  - **Grid Spatial Learner:** Analyzes historical validated merges to learn the typical time and spatial relationships between camera grid cells.
  - **Face Library Builder:** Uses validated, human-confirmed merges to build and refine the library of known faces.

### 11. Inference Manager
- **Function:** A central dispatcher that manages a portfolio of AI inference resources, including local GPUs, remote APIs, and the human review queue.
- **Responsibilities:** Receives generic inference requests (e.g., "run detection," "adjudication"). Selects the best available provider based on priority and health. It treats the human review queue as just another "provider," elegantly integrating the Human-in-the-Loop process.

### 12. Core Data Structures

These data structures are fundamental to managing identity and uncertainty within the system. They are primarily created and managed by the `Track State Manager`.

*   **Agent:**
    *   **Represents:** A persistent, unified identity (e.g., "The Gardener," "Unknown Person #1," "Delivery Truck").
    *   **Contains:** A unique Agent ID, known characteristics (e.g., a face vector for a known person), and a timeline composed of assigned tracks.
    *   **Purpose:** To provide the continuous, long-term history of an entity's movements across all cameras and all time.

*   **Original Track:**
    *   **Represents:** An immutable, continuous observation of an object from a *single camera*. This is the ground-truth record of what a camera saw.
    *   **Contains:** A unique Track ID, the camera ID, start/end times, a path of bounding boxes, pointers to the `previous` and `next` Original Tracks from the same camera, and **a list of references to all `Track Hypotheses` associated with it.**
    *   **Purpose:** To serve as the raw, unalterable evidence of an observation and act as the anchor for all identity hypotheses.

*   **Track Hypothesis (or "Track Copy"):**
    *   **Represents:** A lightweight, disposable link that proposes an `Original Track` belongs to a specific `Agent`.
    *   **Contains:** A reference to the `Original Track` and a reference to the `Agent` it might belong to. It may also contain a confidence score or the specific evidence that generated this hypothesis.
    *   **Purpose:** To allow the system to manage uncertainty. An `Original Track` can have multiple `Track Hypotheses` linking it to different `Agents` simultaneously. When new evidence resolves the uncertainty, the incorrect hypotheses are simply deleted.
