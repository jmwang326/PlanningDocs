# L3: System Orchestration

## Purpose

The Orchestration layer is not a single component but the overarching logic that governs how all the independent processing components (`ChunkProcessor`, `TemporalLinker`, `IdentityResolver`, etc.) interact. It defines the system's data flow, event triggers, and concurrency model, ensuring that raw video data is efficiently and correctly transformed into a coherent timeline.

## Core Philosophy: An Event-Driven System

The system is not a simple, timed pipeline that processes chunks every N seconds. Instead, it is an **event-driven architecture** composed of two primary, interlocking loops. This allows the system to focus resources intelligently, reacting to activity rather than processing empty data.

### 1. The Chunk Processing Pipeline (Activity-Triggered)

This is the main data processing flow that turns video into `LocalTrack`s.

-   **Trigger:** This pipeline is **not** on a fixed timer. It is triggered by a `Camera State Machine` when a camera transitions from the `Active` state to the `Post` state (i.e., when a period of continuous activity has just ended).
-   **Flow:** Once triggered, a 12-second chunk is passed sequentially through the `ChunkProcessor`, `TemporalLinker`, and `IdentityResolver`.
-   **Result:** The pipeline produces `LocalTrack`s that are linked, have an identity (either confirmed or uncertain), and are stored in the database.

### 2. The Evidence Loop (Information-Triggered)

This is the asynchronous loop that refines the system's understanding over time.

-   **Trigger:** This loop is triggered whenever new, external information becomes available. This can be a face recognition result from an AI service, a decision from an LLM, or a manual correction from a human reviewer.
-   **Flow:** The `EvidenceProcessor` uses the new information to resolve an uncertain track. This resolution then triggers a **cascade**, propagating the new knowledge throughout the system and potentially resolving other related uncertainties via a process of elimination.
-   **Result:** The system's timeline becomes more accurate and complete as uncertainties are resolved.

By separating these two loops, the system can process new video footage in near-real-time while continuously improving the accuracy of past events in the background.
