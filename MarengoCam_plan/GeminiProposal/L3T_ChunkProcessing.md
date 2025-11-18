# L3T: Chunk Processing Technical Details

## Component: `ChunkProcessor`

This document provides the specific technical parameters and logic for the `ChunkProcessor` component.

---

## Processing Trigger

-   **Trigger:** The processor is event-driven, not timer-based. It is invoked by the `Camera State Machine` when a camera transitions from `Active` to `Post`.
-   **Input:** A variable-length video segment covering the duration of the `Active` state, plus pre-roll and post-roll buffers.
-   **Overlap:** Overlap is handled logically by the `TemporalLinker` using timestamp continuity, rather than fixed chunk overlaps.

---

## Camera State-Based Inference

The rate and persistence of inference are determined by the camera's current state.

-   **`Standby`**:
    -   **Inference:** Sparse. A motion gate (pixel-diff) is active. YOLO is only run on frames with significant motion.
    -   **Persistence:** No frames are saved.
-   **`Armed`**:
    -   **Inference:** Moderate. YOLO runs to confirm if a candidate track is sustained.
    -   **Persistence:** `sub profile` frames are saved.
-   **`Active`**:
    -   **Inference:** Full. YOLO runs on all frames within the GPU budget.
    -   **Persistence:** Both `sub profile` and `main profile` frames are saved.
-   **`Post`**:
    -   **Inference:** Moderate. YOLO runs to detect if a lost agent re-enters the frame.
    -   **Persistence:** `sub profile` frames are saved.

---

## State Transition Logic

-   **`Standby` -> `Armed`**: The motion gate detects significant pixel delta, and a sparse YOLO run detects a potential object.
-   **`Armed` -> `Active`**: A candidate track is held for ≥ 3 seconds, confirming sustained movement. The track is promoted to a `LocalTrack`.
-   **`Active` -> `Post`**: All `LocalTrack`s on the camera have ended. A post-roll quiet period begins.
-   **`Post` -> `Standby`**: The post-roll timer expires with no new detections.
-   **`Armed` -> `Standby`**: Motion stops before the 3-second threshold is met; the candidate track is discarded.

---

## Filtering Thresholds

-   **Sustained Movement:** A raw YOLO track must persist for **≥ 3 seconds** to be promoted to a `LocalTrack`.
-   **YOLO Confidence Scores:**
    -   `person`: 0.5
    -   `vehicle`: 0.6
    -   `animal`: 0.4

---

## Portal Association Logic

The `ChunkProcessor` is responsible for associating tracks with portals to provide contextual clues for downstream components.

1.  **Load Static Portals:** At startup, load all fixed portal locations (e.g., doors, gates) defined in the system configuration.
2.  **Identify Dynamic Portals:** Identify any `vehicle` tracks that have been stationary for a defined duration (e.g., > 10 seconds). These are treated as temporary portals for the duration of their stationary state.
3.  **Associate Tracks:**
    -   If a `LocalTrack`'s first detection (`entry_cell`) is within a portal's area, set `track.at_portal = portal_id`.
    -   If a `LocalTrack`'s last detection (`exit_cell`) is within a portal's area, set both `track.at_portal = portal_id` and `track.exit_reason = "portal_crossed"`. This explicitly records which portal was entered.

---

## Vehicle Occupancy Logic

**Purpose:** Maintain identity links when agents enter/exit vehicles (mobile portals).

**Logic:**
-   **Vehicle as Portal:** Treat a stationary vehicle as a temporary portal.
-   **Occupancy List:** Maintain a list of agents currently inside each vehicle.
-   **Inference:**
    -   **Exit:** If a vehicle has 1 occupant and 1 person exits, infer identity.
    -   **Ambiguity:** If vehicle has >1 occupant, flag exit track as `UNCERTAIN` with candidates from the occupancy list.
    -   **Trojan Horse:** If a vehicle arrives with unknown occupants, flag initial occupants as `UNCERTAIN` (unknown origin).
