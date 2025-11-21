# L3: GUI & Presentation Layer

## Purpose

This document outlines the functional requirements for user interaction. It defines **what** the user must be able to do, not **how** the interface looks.

## Functional Requirements

### 1. Identity Resolution (The Workbench)
**Goal:** Allow a human to resolve ambiguity that the system cannot handle automatically.

*   **Requirement:** The user must be able to view a list of "Uncertain Tracks" prioritized by importance.
*   **Requirement:** For any uncertain track, the user must be able to compare it against suggested candidates (visual comparison).
*   **Requirement:** The user must be able to execute three distinct decisions:
    1.  **Merge:** Confirm Track A is Agent X.
    2.  **Reject:** Confirm Track A is NOT Agent X.
    3.  **New Agent:** Declare Track A is a new, previously unseen person.

### 2. Forensic Investigation (The Timeline)
**Goal:** Allow a human to reconstruct and verify the history of an agent.

*   **Requirement:** The user must be able to query the history of a specific `GlobalAgent`.
*   **Requirement:** The system must display the sequence of `LocalTracks` that constitute that agent's history.
*   **Requirement:** The user must be able to play the video clip associated with any track.
*   **Requirement:** The user must be able to **Correct Errors**:
    *   **Split:** Break a link between two tracks if the system incorrectly merged them.
    *   **Force Merge:** Manually link two tracks that the system failed to connect.

### 3. Configuration (The World Model)
**Goal:** Allow a human to define the physical constraints of the environment.

*   **Requirement:** The user must be able to define "Portals" (entry/exit zones) on a camera view.
*   **Requirement:** The user must be able to link Portals to logical groups (e.g., "Garage Door" -> "Garage Interior").

### 4. System Monitoring (Health)
**Goal:** Allow a human to know if the system is broken or overloaded.

*   **Requirement:** The user must be able to see the current "Processing Lag" (Time vs. Real-time).
*   **Requirement:** The user must be able to see the size of the "Review Queue".
