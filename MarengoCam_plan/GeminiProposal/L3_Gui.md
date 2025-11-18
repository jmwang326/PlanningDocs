# L3: GUI & Presentation Layer

## Purpose

This document outlines the strategy and structure for the system's user interfaces. The GUI provides the necessary tools for human-in-the-loop interaction, including uncertainty resolution, system monitoring, and timeline analysis.

## UI Strategy: Focused Applications

Instead of a single, monolithic dashboard, the system's UI is composed of several separate, focused web applications. This philosophy allows for simpler development, deployment, and maintenance, and provides a more streamlined user experience for each specific task.

## Primary User Applications

The user interacts with the system primarily through three distinct web-based applications:

1.  **The Review Queue & Panel (The Workbench)**

    **Purpose:** The primary workspace for resolving identity ambiguity.

    ### The Review Queue
    -   **Function:** Displays a prioritized list of uncertain tracks waiting for adjudication.
    -   **Prioritization:** Items are sorted by a score derived from Recency, Simplicity (fewer candidates = higher priority), and Impact (blocking other tracks).
    -   **Consumers:**
        -   **Human Operator:** Uses the GUI to click through items.
        -   **LLM Adjudicator:** Consumes the same queue via API. *Note: The LLM is restricted to visual comparisons only and does not see logical metadata.*

    ### The Review Panel
    -   **Layout:** A side-by-side comparison view.
        -   **Left:** The "Uncertain Track" (6 best crop images).
        -   **Right:** The "Candidate Identity" (6 best crop images from history).
    -   **Evidence Summary:** A human-readable text block explaining *why* these two might match (e.g., "Grid Timing: Strong Match", "Face: Low Confidence").
    -   **Actions:**
        -   `MERGE`: Confirms the link.
        -   `REJECT`: Permanently vetoes this specific link.
        -   `NEW AGENT`: Declares the track as a previously unseen entity.

---

2.  **The Timeline Viewer (The Investigation Tool)**

    **Purpose:** To reconstruct and verify the historical narrative of agents.

    ### Core Features
    -   **Agent Selection:** Searchable list of all GlobalAgents (named and provisional).
    -   **Chronological View:** A vertical list of all `LocalTracks` assigned to the selected agent, stitched into a single timeline.
    -   **Video Playback:** Clicking a track event plays the corresponding video clip.

    ### Advanced Tools
    -   **Force Merge:** A user can select two distinct agents or tracks and execute a "Merge" command, overriding the system's logic.
    -   **Unmerge:** A user can right-click a specific track in a timeline to sever its link to the agent. This action:
        1.  Resets the track to "Uncertain."
        2.  Creates a "Merge Rejection" record to prevent immediate re-merging.
        3.  Returns the track to the Review Queue.
    -   **Zone Occupancy Confirmation:** A tool to clear "ghost" data. The user selects a time range and camera view, sees a list of *all* agents (visible + parking candidates), and confirms "This list is complete." Any unconfirmed agents are logically evicted from the zone/vehicles.

---

3.  **The Portal Editor (The Configuration Tool)**

    **Purpose:** To define the physical world model.

    -   **Interface:** Displays a static reference image for a selected camera.
    -   **Grid Painting:** Overlays the 6x6 grid. Users click/drag to "paint" cells that constitute a portal.
    -   **Metadata:** Users assign a Type (Structure/Vehicle Entry) and a Logical Group (e.g., "The House") to the painted zone.

---

4.  **The Health Dashboard (The Monitor)**

    **Purpose:** To provide real-time visibility into system performance.

    -   **Key Metrics:**
        -   **Processing Lag:** Time difference between "now" and the latest processed track.
        -   **Queue Depth:** Number of pending items in the Review Queue.
        -   **Auto-Merge Rate:** Percentage of tracks resolved without human intervention.
    -   **Alerting:** Visual indicators (Green/Yellow/Red) for metric thresholds.
