# Level 2 - Architecture

## Purpose
This document describes the high-level architecture of the MarengoCam system: what gets processed, who processes it, and why. It outlines the core philosophy, the main software components, and the flow of data between them.

**Related Technical Spec:** [L2T_DataModel.md](L2T_DataModel.md)

---

## Core Philosophy: Manage Video Clips (Tracks)

The fundamental insight of this architecture is that multi-camera surveillance is a **clip management problem**.

The system continuously processes 12-second video chunks from each camera and produces short, independent observations called `LocalTracks`. The entire purpose of the system is to manage these clips by:

1.  **Filtering noise** to ensure we only track sustained, high-confidence movement.
2.  **Linking clips** from the same camera over time to form a continuous history.
3.  **Merging clips** from different cameras that belong to the same person or vehicle.
4.  **Handling uncertainty** gracefully when the evidence is not clear.

Everything else—timelines, agent histories, and forensic analysis—is simply a query or a view on top of this managed collection of `LocalTracks`.

---

## The Seven Components & Data Flow

The system is composed of seven main components that process `LocalTracks` in a pipeline.

```
Video Chunks (every 10s)
    ↓
[1. ChunkProcessor] → LocalTracks (unlinked)
    ↓
[2. TemporalLinker] → LocalTracks (linked on same-camera)
    ↓
[3. IdentityResolver] → LocalTracks (with global_agent_id OR identity_candidates)
    ↓
[4. EvidenceProcessor] → LocalTracks (uncertainty resolved)
    ↓
[5. GUI] → Timeline display, review queue
```

### 1. ChunkProcessor
- **What it does:** Converts 12s video chunks from a camera into a set of `LocalTracks`. Each track corresponds to a unique object detected by the YOLO model in that chunk.
- **Strategic Principle:** *Filter Noise First*. This component applies a ≥3s movement threshold and confidence scoring to filter out insignificant events like shadows or brief glitches, ensuring that only meaningful activity proceeds down the pipeline.
- **See:** [L3_ChunkProcessing.md](L3_ChunkProcessing.md)

### 2. TemporalLinker
- **What it does:** Links `LocalTracks` from the same camera across consecutive chunks.
- **Strategic Principle:** *Link Tracks Over Time*. Because YOLO may assign a new tracking ID in each chunk, this component uses a 2-second overlap between chunks to match objects by position and appearance, creating a continuous history on a single camera.
- **See:** [L3_TemporalLinking.md](L3_TemporalLinking.md)

### 3. IdentityResolver
- **What it does:** Assigns a `global_agent_id` to `LocalTracks` to merge observations from *different cameras* into a single agent history.
- **Strategic Principle:** *Merge Across Cameras Using an Evidence Hierarchy*. This is the core of cross-camera tracking. It applies a strict, ordered set of rules:
    1.  **Face Recognition:** A strong match is definitive and results in an auto-merge.
    2.  **Grid Logic:** Plausible movement between camera grid zones (based on learned travel times) with no conflicting evidence also results in an auto-merge.
    3.  **Ambiguity:** If evidence is weak or contradictory, the track is not merged. Instead, it is marked with a set of `identity_candidates`.
- **See:** [L3_IdentityResolution.md](L3_IdentityResolution.md)

### 4. EvidenceProcessor
- **What it does:** Resolves tracks that the `IdentityResolver` marked as uncertain.
- **Strategic Principle:** *Accept Uncertainty*. This component understands that a decision should not be forced without sufficient evidence. It waits for new information—such as a clearer face image, a human review decision, or an LLM adjudication—to resolve the `identity_candidates`. This prevents the system from making incorrect merges that corrupt the timeline.
- **See:** [L3_EvidenceProcessing.md](L3_EvidenceProcessing.md)

### 5. GUI
- **What it does:** Provides the user interface for viewing agent timelines, playing back event clips, and reviewing uncertain merges.
- **Strategic Principle:** The GUI is the primary tool for the *Human-in-the-Loop* process, allowing for the bootstrapping and correction of the system's automated decisions.
- **See:** [L3_Gui.md](L3_Gui.md)

### 6. Configuration
- **What it does:** Manages all system settings, such as camera details, portal definitions, and evidence thresholds, from a central YAML file.
- **See:** [L3_Configuration.md](L3_Configuration.md)

### 7. SystemHealth
- **What it does:** Monitors system performance, including GPU utilization, processing latency, and queue depths, to ensure reliability.
- **See:** [L3_SystemHealth.md](L3_SystemHealth.md)
