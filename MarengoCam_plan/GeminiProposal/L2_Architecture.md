# L2: System Architecture

## Core Philosophy: The Hub-and-Spoke Model

The system is not a simple linear pipeline. It is an intelligent, event-driven system orchestrated by a central **Conductor**. This "Hub-and-Spoke" architecture allows for complex decision-making, asynchronous processing, and a clear separation of concerns.

The **Conductor** is not a single component but a distributed system of event-driven components that communicate with each other through a well-defined set of events and state transitions.

## The Hub

The **Hub** is the central data store, the `Database`, which holds the state of the system. It is the single source of truth for all components.

## The Spokes

The **Spokes** are the individual processing components that perform specific tasks. They are triggered by events and communicate with each other through the **Hub**.

The main spokes are:

*   **ChunkProcessor:** Converts a raw video chunk into a set of filtered, significant `LocalTracks`.
*   **TemporalLinker:** Links `LocalTracks` from the same camera across consecutive time chunks.
*   **IdentityResolver:** Assigns a global identity (`global_agent_id`) to `LocalTracks` from different cameras.
*   **EvidenceProcessor:** Resolves identity uncertainty in `LocalTracks`.
*   **GUI:** Provides the user interface for system interaction and human-in-the-loop review.

## The Conductor

The **Conductor** is the collection of rules and triggers that orchestrate the flow of data between the **Spokes**. It is implemented as a set of state machines and event listeners that react to changes in the **Hub**.

The key responsibilities of the **Conductor** are:

*   **Triggering the pipeline:** The `Camera State Machine` monitors motion and triggers the `ChunkProcessor` when a significant event occurs.
*   **Orchestrating the processing flow:** The **Conductor** ensures that the `LocalTracks` are processed by the `TemporalLinker`, `IdentityResolver`, and `EvidenceProcessor` in the correct order.
*   **Managing the evidence cascade:** The **Conductor** manages the feedback loop that is triggered when new evidence is available, ensuring that the new information is propagated throughout the system.

## System Data Flow

The system data flow is not a simple linear pipeline. Instead, it is a series of event-driven interactions between the **Spokes**, orchestrated by the **Conductor**.

1.  The `Camera State Machine` detects a significant event and triggers the `ChunkProcessor`.
2.  The `ChunkProcessor` creates new `LocalTracks` and stores them in the `Database`.
3.  The new `LocalTracks` trigger the `TemporalLinker`, which links them to existing tracks.
4.  The linked tracks trigger the `IdentityResolver`, which assigns a `global_agent_id` or a list of `identity_candidates`.
5.  If new evidence becomes available (e.g., from the `GUI` or an external system), the `EvidenceProcessor` is triggered, which can lead to a cascade of updates and re-resolutions.