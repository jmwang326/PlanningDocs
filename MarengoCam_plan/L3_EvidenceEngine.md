# Level 3 - Evidence Engine - Tactical

## 1. Purpose

The **Evidence Engine** is a specialized data aggregation component. Its sole purpose is to gather and package all available pieces of evidence for a potential merge between two tracks. It operates as a fact-finding service, providing a structured "Evidence Report" to the `Track Merging Engine`, which then makes the final decision.

## 2. Key Responsibilities

- **Evidence Aggregation:** For a given pair of tracks (a source and a candidate destination), the engine is responsible for gathering all possible points of comparison.

- **Interface with Inference Manager:** The engine is a primary client of the `Inference Manager`. It will make asynchronous requests for computationally expensive evidence, such as:
    - `face_similarity`: Comparing the best face crops from each track.
    - `reid_similarity`: Comparing the Re-ID embeddings of the agents.
    - `color_similarity`: Comparing the dominant clothing colors.

- **Static Data Lookups:** It performs fast lookups against the system's configuration and state, such as:
    - **Portal Check:** Does a configured portal connect the exit cell of the source track and the entry cell of the destination track?
    - **Timing Analysis:** Is the time delta between the tracks within the plausible window defined for that portal (or the general discovery window)?

- **Evidence Report Generation:** It compiles all gathered facts—both positive and negative—into a single, structured `EvidenceReport` object. This report does not contain a decision, only the evidence itself.

## 3. Operational Flow

1.  **Request:** The `Track State Manager` identifies a potential merge and asks the `Track Merging Engine` to evaluate it.
2.  **Delegation:** The `Track Merging Engine` immediately delegates the fact-finding task to the `Evidence Engine`, passing it the two track IDs.
3.  **Parallel Evidence Gathering:** The `Evidence Engine` initiates multiple evidence-gathering tasks in parallel:
    - It makes one or more asynchronous calls to the `Inference Manager` for AI-based evidence (face, Re-ID).
    - Simultaneously, it performs the fast, local checks (portal, timing).
4.  **Compilation:** As the results return, the engine compiles them into an `EvidenceReport`.
5.  **Response:** Once all evidence has been gathered (or has timed out), the `Evidence Engine` returns the complete `EvidenceReport` to the `Track Merging Engine`.

## 4. Strategic Importance

By isolating fact-finding from decision-making, the system becomes far more modular. We can add new types of evidence (e.g., "gait analysis") by simply updating the `Evidence Engine` and adding a new rule to the `Track Merging Engine`, without touching any other part of the system. This separation makes the logic cleaner, easier to test, and more extensible. The `Track Merging Engine` remains a simple, readable rules engine, while the `Evidence Engine` handles the complexity of asynchronous I/O and data aggregation.
