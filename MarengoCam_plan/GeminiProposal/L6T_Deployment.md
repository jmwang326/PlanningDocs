# L6T: Deployment Technical Roadmap

This document provides the detailed, phase-by-phase technical roadmap for the MarengoCam project. It outlines which components are created or evolved at each stage, along with their specific success gates.

---

### Phase 0: The Foundation (Data Ingestion)

-   **Goal:** Prove reliable data acquisition and storage.
-   **New Components:**
    -   `FrameFetcher`: Acquires images from camera sources.
    -   `FrameStore`: Saves image files to a persistent volume.
    -   `ObjectDetector`: Runs YOLO model on images to produce raw detections.
-   **Evolving Components:**
    -   **`Conductor (V1)`:** A basic script to orchestrate the flow: Fetch -> Detect -> Save Detections to DB.
    -   **`Database Schema (V1)`:** Initial tables for `Cameras` and raw `Detections`.
-   **Success Gate:** Raw object detections are continuously and reliably written to the database for all cameras.

---

### Phase 1: Single-Camera Tracking

-   **Goal:** Create coherent `LocalTrack`s on individual cameras.
-   **New Components:**
    -   `ChunkProcessor`: Groups detections into temporal chunks.
    -   `TemporalLinker`: Links detections across chunks to form tracks.
    -   `CameraStateMachine`: Manages the operational state of each camera.
-   **Evolving Components:**
    -   **`Conductor (V2)`:** Evolves into a pipeline manager, orchestrating the state machine and the chunk/link process.
    -   **`Database Schema (V2)`:** The `LocalTracks` table is added.
-   **Success Gate:** A person walking across a single camera view produces one continuous `LocalTrack` record in the database.

---

### Phase 2: The Human-and-AI-in-the-Loop

-   **Goal:** Build the core review loop for both human and AI adjudicators.
-   **New Components:**
    -   `ReviewQueue`: Manages the list of uncertain decisions.
    -   `LLMAdjudicator`: An automated service for simple visual comparisons.
    -   **`GUI (V1 - Developer Tool)`**:
        -   `ReviewPanel (V1)`: Functional, side-by-side image comparison.
        -   `TimelineViewer (V1)`: Text-based list to verify merges.
-   **Evolving Components:**
    -   **`Conductor (V3)`:** Becomes the master orchestrator, managing the main "fetch-delegate-execute" loop for identity resolution.
    -   **`IdentityResolver (V1)`:** A stub that finds any plausible candidate and creates a `ReviewQueue` entry.
    -   **`EvidenceProcessor (V1)`:** Processes merge commands from both human and LLM sources.
    -   **`Database Schema (V3)`:** Adds `GlobalAgents` and `ReviewQueueEntries` tables.
-   **Success Gate:** A track can be successfully merged across two cameras by either a human or the LLM, and the result is verifiable in the `TimelineViewer`.

---

### Phase 3: Introducing Intelligence

-   **Goal:** Begin automating the majority of merge decisions.
-   **New Components:**
    -   `GridLearning`: Passively learns travel times from validated merges.
    -   `ExternalAI`: A client for the face recognition service.
    -   **`GUI (V2 - Analyst Tool)`**:
        -   `FaceLibraryManager (V1)`: UI to assign names to faces.
        -   `TimelineViewer (V2)`: Adds a video player and displays merge reasons.
-   **Evolving Components:**
    -   **`IdentityResolver (V2)`:** Major upgrade. Adds logic to use Face and Grid data to perform high-confidence auto-merges, bypassing the review queue.
    -   **`EvidenceProcessor (V2)`:** Gains the new responsibility of feeding merge data to the `GridLearning` component.
-   **Success Gate:** A known person is automatically tracked across cameras with no human/LLM intervention.

---

### Phase 4: Advanced Logic

-   **Goal:** Handle complex, real-world scenarios involving portals and vehicles.
-   **New Components:**
    -   `PortalTransitions`: Logic for fixed portals.
    -   `VehicleOccupancy`: Logic for mobile portals (vehicles).
    -   `ProcessOfElimination`: Logic for resolving ambiguity via cascades.
    -   **`GUI (V3 - Admin Tool)`**:
        -   `Portal Editor (V1)`: GUI for "painting" portal zones on a grid.
-   **Evolving Components:**
    -   **`Conductor (V4)`:** Upgraded to manage hierarchical/embedded portals ("car in garage").
    -   **`IdentityResolver (V3)`:** Upgraded to incorporate portal and vehicle logic into its evidence hierarchy.
    -   **`Database Schema (V4)`:** Adds `Portals` and `GridLearnings` tables.
-   **Success Gate:** The "Garage Puzzle" and "Ambiguous Portal Exit" stress tests pass successfully.

---

### Phase 5: Full Autonomy & Monitoring

-   **Goal:** Transition to a fully autonomous system and provide tools to monitor its health.
-   **New Components:**
    -   `SystemHealth`: A service to collect and aggregate system metrics.
    -   **`GUI (V-Final - User Application)`**:
        -   `HealthDashboard (V1)`: Displays real-time system vitals.
        -   Final polish and UX improvements across all components.
-   **Evolving Components:**
    -   **`Conductor (V5)`:** Adds the final responsibility of managing background tasks, such as triggering the `SystemHealth` collector.
-   **Success Gate:** The `HealthDashboard` is live and accurately reflects the state of the running system, showing a high auto-merge rate and low review queue depth.
