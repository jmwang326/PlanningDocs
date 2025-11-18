# Level 2 - Architecture

## Purpose
This document describes the high-level architecture of the MarengoCam system using the Component-Contract Specification style. It outlines the core philosophy, the main software components, and the flow of data between them.

**Related Technical Spec:** [L2T_DataModel.md](L2T_DataModel.md)

---

## Core Philosophy

The architecture treats multi-camera surveillance as a **clip management problem**. The system's primary function is to manage `LocalTracks` (short video observations) through four sequential operations:

1.  **Filter:** Remove noise and insignificant events.
2.  **Link:** Connect `LocalTracks` from the same camera over time.
3.  **Merge:** Associate `LocalTracks` from different cameras that belong to the same entity.
4.  **Resolve:** Handle uncertainty using new evidence or human review.

All user-facing features (e.g., timelines, analysis) are views generated from the resulting collection of managed tracks.

---

## System Data Flow

The system processes data in a pipeline, where the output of one component becomes the input for the next.

1.  `Video Chunks` are sent to the **ChunkProcessor**.
2.  The resulting `LocalTracks` (unlinked) are sent to the **TemporalLinker**.
3.  The `LocalTracks` (now linked) are sent to the **IdentityResolver**.
4.  The `LocalTracks` (now with identity information or candidates) are sent to the **EvidenceProcessor** for final resolution.
5.  The **GUI** consumes the processed data to display timelines and review queues for the user.

---

## Component Contracts

### 1. ChunkProcessor
**Purpose:** Converts a raw video chunk into a set of filtered, significant `LocalTracks`.
**Inputs:**
- `VideoChunk` (12s video segment)
**Process:**
1.  Detect all objects using the YOLO model.
2.  Apply a ≥3s movement threshold to filter transient objects.
3.  Apply confidence scoring to filter low-confidence detections.
4.  Generate a `LocalTrack` for each significant object.
**Outputs:**
- `List[LocalTrack]` (unlinked)
**Specification:** [L3_ChunkProcessing.md](L3_ChunkProcessing.md)

### 2. TemporalLinker
**Purpose:** Links `LocalTracks` from the same camera across consecutive time chunks.
**Inputs:**
- `List[LocalTrack]` (from a single camera)
**Process:**
1.  Match objects between consecutive chunks based on position and appearance.
2.  Create a continuous history for each object on the same camera.
**Outputs:**
- `List[LocalTrack]` (linked by `track_id` across time)
**Specification:** [L3_TemporalLinking.md](L3_TemporalLinking.md)

### 3. IdentityResolver
**Purpose:** Assigns a global identity (`global_agent_id`) to `LocalTracks` from different cameras.
**Inputs:**
- `List[LocalTrack]`
**Process:**
1.  Attempt to merge tracks using a strict evidence hierarchy (Face > Grid Logic).
2.  If a high-confidence match is found, assign `global_agent_id` and merge.
3.  If evidence is weak or contradictory, assign a list of `identity_candidates` instead of merging.
**Outputs:**
- `List[LocalTrack]` (with `global_agent_id` or `identity_candidates`)
**Specification:** [L3_IdentityResolution.md](L3_IdentityResolution.md)

### 4. EvidenceProcessor
**Purpose:** Resolves identity uncertainty in `LocalTracks`.
**Inputs:**
- `List[LocalTrack]` (with `identity_candidates`)
**Process:**
1.  Awaits new information (e.g., clearer face image, human review, LLM decision).
2.  Uses the new evidence to resolve the `identity_candidates` to a single `global_agent_id`.
**Outputs:**
- `List[LocalTrack]` (with uncertainty resolved)
**Specification:** [L3_EvidenceProcessing.md](L3_EvidenceProcessing.md)

### 5. GUI
**Purpose:** Provides the user interface for system interaction and human-in-the-loop review.
**Inputs:**
- `List[LocalTrack]` (fully processed)
- `List[TimelineEvent]`
**Process:**
1.  Display agent timelines.
2.  Present uncertain merges in a review queue.
3.  Accept user decisions to resolve uncertainty.
**Outputs:**
- `UserDecision` (fed back to EvidenceProcessor)
**Specification:** [L3_Gui.md](L3_Gui.md)

### 6. Configuration
**Purpose:** Manages all system settings from a central source.
**Inputs:**
- `config.yaml`
**Process:**
1.  Parses the configuration file.
2.  Provides settings (e.g., camera details, thresholds) to all other components.
**Outputs:**
- `SystemConfig` object
**Specification:** [L3_Configuration.md](L3_Configuration.md)

### 7. SystemHealth
**Purpose:** Monitors system performance and reliability.
**Inputs:**
- Logs and metrics from all components (e.g., GPU util, latency, queue depth).
**Process:**
1.  Aggregate metrics.
2.  Provide a dashboard or alerts on system status.
**Outputs:**
- `HealthStatus` report
**Specification:** [L3_SystemHealth.md](L3_SystemHealth.md)
