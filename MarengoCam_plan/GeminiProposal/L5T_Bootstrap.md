# L5T: Bootstrap Technical Details

This document provides the technical specifications for the bootstrap phase, including the phased workflow, required tooling, and transition metrics.

---

## Dependency Manifest

-   **L3_Configuration.md**: Defines the `PROD_LEARNING` and `PROD_AUTO` operational stages.
-   **L3_Gui.md**: General overview of the GUI approach.
-   **L3T_EvidenceProcessing.md**: Specifies the review queue logic.
-   **L3T_Gui.md**: Specifies the review panel interface.
-   **L3T_IdentityResolution.md**: Details the grid learning mechanism.

---

## Phased Bootstrap Workflow

### Phase 1: Initial Data Collection (Approx. Days 1-14)

-   **System State:** `stage: PROD_LEARNING`
    -   `identity_resolution.face_match_threshold`: High (e.g., 0.85) to prevent false positives.
    -   All uncertain merges are sent to the Review Queue.
    -   Grid learning populates from human-validated merges only.
-   **Human Operator Workflow:**
    -   **Primary Goal:** Identify 5-10 key individuals in the Face Library Manager.
    -   Perform 50-100 reviews per day in the Review Queue.
    -   For each key person, add 50-100 high-quality face crops from multiple cameras and lighting conditions.

### Phase 2: Accelerated Learning (Approx. Days 15-30)

-   **System State:** `stage: PROD_LEARNING` (continued)
    -   System begins to auto-merge tracks based on high-confidence face matches from the growing library.
    -   Grid learning accelerates as more paths are validated.
-   **Human Operator Workflow:**
    -   Review load decreases as the system takes over simple merges.
    -   Focus shifts to reviewing more complex cases (occlusions, groups).
    -   Use the Re-ID Crop Exporter to generate initial training sets.

### Phase 3: Transition to Autonomous Operation

-   **Trigger:** The system meets the `Transition Metrics` defined below.
-   **System State:** `stage: PROD_AUTO`
    -   `identity_resolution.face_match_threshold`: Lowered (e.g., 0.70) for more aggressive matching.
    -   LLM-based adjudication may be enabled for uncertain cases.
-   **Human Operator Workflow:**
    -   Review load drops to a minimum (e.g., < 10 complex cases per day).
    -   Focus on system health monitoring and periodic curation of training data.

---

## Tooling Specifications

### 1. Review Queue & Panel

-   **Reference:** `L4_ReviewQueue.md`, `L4_ReviewPanel.md`
-   **Functionality:**
    -   Presents a prioritized list of uncertain `LocalTrack` merges.
    -   Provides a side-by-side comparison UI.
    -   Allows operator to `merge`, `reject`, or `create new agent`.
    -   **Bootstrap Feature:** Must include a one-click action to "Send selected face crops to Face Library Manager" for the relevant agent.

### 2. Face Library Manager

-   **Reference:** (Requires new `L4_FaceLibrary.md`)
-   **Functionality:**
    -   Provides a gallery of all face crops, grouped by `AgentID`.
    -   Allows operator to assign/change agent names (e.g., "John Doe").
    -   Allows operator to add/remove individual face samples from an agent's library.
    -   Provides a dashboard to visualize coverage per agent (e.g., heatmap of cameras vs. lighting conditions).
    -   Includes a button to trigger the `retrain` command on the external face recognition service (e.g., CodeProject.AI).

### 3. Re-ID Crop Exporter

-   **Implementation:** A command-line interface (CLI) or simple web UI.
-   **Functionality:**
    -   Filters for tracks associated with validated merges.
    -   Allows selection of agents.
    -   Exports all associated, high-quality crops into a structured directory format suitable for model training.
-   **Output Format:**
    ```
    /reid_training_data/
      <export_timestamp>/
        manifest.json
        person_<agent_id>/
          <track_id>_<timestamp>_<camera_id>.jpg
          ...
        vehicle_<agent_id>/
          ...
    ```

---

## Transition Metrics

The transition from `PROD_LEARNING` to `PROD_AUTO` is authorized when the following metrics, exposed via the `SystemHealth` component, are met and sustained for 3 consecutive days:

-   **`auto_merge_rate_percent`**: `> 80.0`
-   **`face_library.known_agents`**: `>= 5`
-   **`face_library.high_coverage_agents`**: `>= 3` (High coverage = >50 crops across >2 cameras).
-   **`grid_learning.path_coverage_percent`**: `> 80.0` (for paths seen more than 10 times).
