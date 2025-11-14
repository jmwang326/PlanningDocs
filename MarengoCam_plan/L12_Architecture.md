# Level 12 - System Architecture

## Purpose
This document outlines the high-level technical components (core processes) of the MarengoCam system. These components are the technical implementation of the strategies defined in `L2_Strategy.md`. Each component represents a major functional block of the system.

## Core Real-Time Components

### 1. Frame Acquisition (Blue Iris Integration)
- **Function:** Fetches JPEG snapshots from Blue Iris HTTP endpoints at configured frame rate (10 FPS production, 2 FPS DEV_SLOW).
- **Responsibilities:** Polls Blue Iris `/image/{short}?q=60` endpoint for sub and main profiles, handles connection errors, maintains 25s rolling frame buffer for chunked processing and post-roll.
- **Corresponds to L3 Tactic:** `L3_VideoIngestor.md`

### 2. Detection & Intra-Camera Tracking
- **Function:** Processes video chunks using the object detection and tracking model (YOLO11).
- **Responsibilities:** Identifies objects (people, vehicles) in frames and links them across frames within a single camera's view to create initial "tracklets." Filters out initial noise based on persistence and movement.
- **Corresponds to L3 Tactic:** `L3_DetectionAndTracking.md`

### 3. Track State Manager
- **Function:** The central state machine for the entire system.
- **Responsibilities:** Manages the lifecycle of all tracks and agents. Tracks states include `new`, `active`, `lost`, `uncertain_merge`, `merged`, and `archived`. This component orchestrates the flow of data between the other components.
- **Corresponds to L3 Tactic:** `L3_TrackStateManager.md`

### 4. Evidence Engine
- **Function:** Gathers all possible evidence for potential track merges.
- **Responsibilities:** For any two tracks, it calculates face recognition similarity, spatial/temporal proximity, visual similarity (e.g., clothing color), and checks for portal crossings. It provides this evidence package to the Merging Engine.
- **Corresponds to L3 Tactic:** `L3_EvidenceEngine.md`

### 5. Track Merging Engine
- **Function:** Decides whether to merge two or more tracklets into a single "agent" timeline.
- **Responsibilities:** Applies the evidence hierarchy defined in `L2_Strategy.md`. Executes merges based on definitive evidence (face match) or weighted evidence, and flags merges that are uncertain.
- **Corresponds to L3 Tactic:** `L3_TrackMerging.md`

### 6. Timeline Reconstruction
- **Function:** Converts the raw data of merged agent tracks into a human-readable narrative.
- **Responsibilities:** Generates descriptive event logs (e.g., "Person A entered via East Gate at 10:32 AM") from the agent's path through camera zones and portals.
- **Corresponds to L3 Tactic:** `L3_TimelineReconstruction.md`

### 7. Panel Assembly Service
- **Function:** A specialized, stateless image processing service.
- **Responsibilities:** Creates standardized visual artifacts (e.g., 2x6 comparison panels) for review. It handles all image fetching, cropping, resizing, upscaling, and composition, returning only a file path to the final image.
- **Corresponds to L3 Tactic:** `L3_PanelAssemblyService.md`

### 8. Presentation Layer (GUI)
- **Function:** The user-facing interface for viewing and interacting with the system's output.
- **Responsibilities:** Displays the reconstructed agent timelines, allows users to review video clips, and provides the interface for the "Human-in-the-Loop" review process by presenting tasks from the review queue.
- **Corresponds to L3 Tactic:** `L3_Gui.md`

## Supporting Systems

### 9. Configuration System
- **Function:** Provides configuration data to all other components.
- **Responsibilities:** Manages camera settings, portal definitions, detection thresholds, and evidence tuning parameters. This is the system's central control panel.
- **Corresponds to L3 Tactic:** `L3_Configuration.md`

### 10. Portals
- **Function:** Portals are a core concept representing a defined transition point where an object can move between camera views or effectively exit the visible space.
- **Responsibilities:** They are defined in the `Configuration System`. A portal crossing is a key piece of evidence used by the `Evidence Engine` and `Track Merging Engine`.
- **Clarification:** A portal is not just a redundant path between two camera zones. It can also represent a location where an object can be legitimately hidden for a period (e.g., entering a building, a garage, or a dense thicket). This allows the system to tolerate longer temporal gaps in tracking without losing the agent's identity, as the "time spent in portal" is a justifiable, albeit unobserved, part of the agent's journey.

### 11. Learning & Training Subsystem
- **Function:** Offline processes responsible for improving the system's accuracy over time.
- **Sub-components:**
  - **Grid Spatial Learner:** Analyzes historical validated merges to learn the typical time and spatial relationships between camera grid cells.
  - **Face Library Builder:** Uses validated, human-confirmed merges to build and refine the library of known faces.

### 12. Inference Manager
- **Function:** A central dispatcher that manages a portfolio of AI inference resources, including local GPUs, remote APIs, and the human review queue.
- **Responsibilities:** Receives generic inference requests (e.g., "run detection," "adjudication"). Selects the best available provider based on priority and health. It treats the human review queue as just another "provider," elegantly integrating the Human-in-the-Loop process.
- **Corresponds to L3 Tactic:** `L3_InferenceManager.md`

### 13. Core Data Structures

These data structures, defined in the `L13_Database.md` schema, are fundamental to managing identity and uncertainty. They are primarily created and managed by the `Track State Manager`.

*   **Agent:**
    *   **Represents:** A persistent, unified identity (e.g., "The Gardener," "Unknown Person #1," "Delivery Truck").
    *   **Contains:** A unique `agent_id` and links to known characteristics (e.g., an `identity_id` if the person is named).
    *   **Purpose:** To group multiple `Track` observations into a single, continuous history of an entity's movements across all cameras and all time.

*   **Track:**
    *   **Represents:** A continuous observation of an object from a *single camera*.
    *   **Contains:** A unique `track_id`, the `camera` it was on, start/end times, and a direct link via `agent_id` to the `Agent` it currently belongs to.
    *   **Handling Uncertainty:** To manage ambiguity, a `Track` also has a `candidate_agents` field. If the system is unsure which `Agent` a `Track` belongs to, it can hold a list of possibilities here until the uncertainty is resolved. This is a pragmatic approach to avoid making incorrect permanent assignments.
