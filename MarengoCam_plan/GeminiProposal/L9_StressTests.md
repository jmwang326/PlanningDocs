# L9: Stress Test Scenarios

## Component: `SystemStressTests`

## Purpose

This document outlines a series of complex, real-world scenarios designed to test the limits of the system's logic, particularly the interaction between its most advanced components. These are not simple unit tests; they are holistic "end-to-end" challenges that simulate the messy, ambiguous nature of reality. The goal is to ensure the system behaves gracefully and logically under pressure.

## High-Level Scenarios

### The "Gardener" Endurance Test

-   **Objective:** To test long-term, intermittent tracking and the system's ability to handle multi-agent interactions.
-   **Scenario:**
    1.  An unknown individual (the "Gardener") arrives and works on the property for 2-3 hours, frequently entering and exiting camera views.
    2.  The system must correctly identify this individual as a single, persistent (though unknown) agent across dozens of separate tracks without human intervention, relying on LLM adjudication.
    3.  During this period, the Gardener interacts with two known residents in the same frame for several minutes.
    4.  A delivery driver (another unknown agent) arrives and briefly interacts with the Gardener.
-   **Success Criteria:**
    -   The Gardener is correctly maintained as a single `GlobalAgent` throughout the entire duration.
    -   The interactions with residents and the driver are recorded as concurrent, overlapping tracks, not confused or merged events.
    -   A complete and accurate timeline can be retrieved for the Gardener, the residents, and the delivery driver, with no data corruption.

### The "Garage Puzzle" Logic Test

-   **Objective:** To test the system's logical inference, particularly the interplay between Portals, Vehicle Occupancy, and the Process of Elimination.
-   **Scenario:**
    1.  Four known agents (P1, P2, P3, P4) are known to be inside the "House" portal group.
    2.  They all move into the "Garage" portal.
    3.  A known vehicle (`V1`) emerges from the Garage.
    4.  Two agents (`P1`, `P2`) then emerge from the Garage *on foot* and re-enter the House.
-   **Success Criteria:**
    -   The system must correctly update the state of `V1`'s potential occupants to `{P3, P4}`.
    -   If `V1` later appears at another camera and a single person exits, the system must correctly identify that the list of candidates for this new track is only `{P3, P4}`, making the review trivial.

### The "Party" Saturation Test

-   **Objective:** To test the system's performance and stability under high-load conditions.
-   **Scenario:**
    1.  Ten to fifteen people (a mix of known and unknown) are present across multiple camera views simultaneously for one hour.
    2.  Agents form and break small groups, creating constant occlusions and concurrent track scenarios.
    3.  Multiple portal transitions (in and out of the "House" portal) and vehicle movements occur.
-   **Success Criteria:**
    -   The `ProcessingLag` on the `HealthDashboard` remains within acceptable limits (e.g., < 30 seconds).
    -   The `ReviewQueue` depth remains manageable and does not grow exponentially.
    -   The system does not crash or suffer from deadlocks.
    -   Timelines for known residents remain accurate despite the chaotic environment.
