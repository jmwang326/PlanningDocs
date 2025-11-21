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
    5.  **The "Bathroom Break" (Chicken & Egg):** The Gardener disappears into a portal (e.g., Shed) for 15 minutes. This exceeds the grid's temporal logic. The system creates a new agent ID upon exit.
-   **Success Criteria:**
    -   The Gardener is correctly maintained as a single `GlobalAgent` throughout the entire duration.
    -   **LLM Resolution:** The LLM correctly identifies the "Pre-Shed" and "Post-Shed" agents as the same person based on visual similarity, merging the broken timeline.
    -   The interactions with residents and the driver are recorded as concurrent, overlapping tracks, not confused or merged events.
    -   A complete and accurate timeline can be retrieved for the Gardener, the residents, and the delivery driver, with no data corruption.

### The "Garage Puzzle" Logic Test

-   **Objective:** To test the system's logical inference, particularly the interplay between Portals, Vehicle Occupancy, and the Process of Elimination.
-   **Scenario:**
    1.  Four known agents (P1, P2, P3, P4) are known to be inside the "House" portal group.
    2.  They all move into the "Garage" portal.
    3.  A known vehicle (`V1`) emerges from the Garage.
    4.  Two agents (`P1`, `P2`) then emerge from the Garage *on foot* and re-enter the House.
    5.  **The "Trojan Horse" (Chicken & Egg):** `V1` drives away and returns 20 minutes later. The system must remember that `V1` still contains `{P3, P4}`. When a single person exits `V1`, the system must identify them as either P3 or P4, not a stranger.
-   **Success Criteria:**
    -   The system must correctly update the state of `V1`'s potential occupants to `{P3, P4}`.
    -   If `V1` later appears at another camera and a single person exits, the system must correctly identify that the list of candidates for this new track is only `{P3, P4}`, making the review trivial.

### The "Party" Saturation Test

-   **Objective:** To test the system's performance and stability under high-load conditions.
-   **Scenario:**
    1.  Ten to fifteen people (a mix of known and unknown) are present across multiple camera views simultaneously for one hour.
    2.  Agents form and break small groups, creating constant occlusions and concurrent track scenarios.
    3.  Multiple portal transitions (in and out of the "House" portal) and vehicle movements occur.
    4.  **Queue Saturation (Chicken & Egg):** The Review Queue exceeds 100 items. The system stops learning because merges aren't being validated fast enough.
-   **Success Criteria:**
    -   The `ProcessingLag` on the `HealthDashboard` remains within acceptable limits (e.g., < 30 seconds).
    -   **Degraded Mode:** The system successfully enters a "Survival Mode" (e.g., auto-archiving low-confidence tracks) to prevent a crash.
    -   The system does not crash or suffer from deadlocks.
    -   Timelines for known residents remain accurate despite the chaotic environment.

### The "Sensor Blackout" Test (Graceful Failure)

-   **Objective:** To test the system's resilience to hardware failure and missing data.
-   **Scenario:**
    1.  An agent is tracked entering a zone covered by `Camera_X`.
    2.  `Camera_X` loses power or network connection immediately after entry.
    3.  The agent walks through the zone and appears on `Camera_Y` 30 seconds later.
-   **Success Criteria:**
    -   **Broken Chain:** The system does *not* attempt to "teleport" the agent from the previous camera to `Camera_Y` using the Grid, because the intermediate link is missing.
    -   **New Agent:** The system correctly treats the appearance on `Camera_Y` as a "New Agent" (or uncertain candidate).
    -   **Alerting:** The system flags the `Camera_X` outage but continues processing other cameras without crashing.

### The "Hot Add" Configuration Test

-   **Objective:** To test the system's ability to adapt to configuration changes without a full restart.
-   **Scenario:**
    1.  The system is running in `PROD_AUTO` mode.
    2.  A user edits `cameras.yaml` to add a new camera (`Camera_Z`) and defines its stream URL.
    3.  The user saves the file.
-   **Success Criteria:**
    -   **Detection:** The `Configuration` component detects the file change and validates the new schema.
    -   **Orchestration:** The `Conductor` spins up a new `FrameFetcher` and `ChunkProcessor` task for `Camera_Z`.
    -   **Grid Expansion:** The `IdentityResolver` initializes empty grid rows/columns for `Camera_Z` without crashing existing lookups.
    -   **No Downtime:** Existing tracks on other cameras are not interrupted.
