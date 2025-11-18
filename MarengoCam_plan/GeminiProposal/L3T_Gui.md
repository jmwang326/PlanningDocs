# L3T: GUI Technical Details

This document provides the specific technical details for the GUI and presentation layer.

---

## Technology Stack

-   **Backend API:** A Python-based framework such as **FastAPI** or **Flask** will serve the REST API.
-   **Frontend:** A modern JavaScript framework such as **React** or **Vue.js** will be used to build the single-page applications.
-   **Real-time Communication:** **WebSockets** will be used to push real-time updates to the Health Dashboard and Review Queue.
-   **CLI Tools:** The **Click** or **Typer** library will be used for building command-line interface tools.

---

## API Architecture

The system exposes a RESTful API to be consumed by the frontend applications.

### Key Endpoint Groups

-   **/api/queue/**
    -   `GET /`: Fetch the prioritized list of items in the review queue.
    -   `GET /{track_id}/panel`: Get the detailed data needed to render the review panel for a specific track.
    -   `POST /{track_id}/review`: Submit a human or LLM decision for a track.

-   **/api/agents/**
    -   `GET /`: Get a list of all known `GlobalAgent`s.
    -   `GET /{agent_id}/timeline`: Fetch the chronological list of `LocalTrack`s for a specific agent.

-   **/api/tracks/**
    -   `GET /{track_id}/video`: Stream the video for a specific `LocalTrack`.

-   **/api/health/**
    -   `GET /`: Get the current real-time status of all system components.
    -   `GET /metrics`: Get historical time-series data for key performance indicators.
    -   `GET /alerts`: Get a list of all active and recent system alerts.

-   **/api/config/**
    -   `GET /{section}`: Get the current configuration for a specific section (e.g., `cameras`, `portals`).
    -   `POST /{section}`: Add or update a configuration item.

---

## Development Phases

The GUI components will be developed in phases to align with the overall system bootstrap and refinement process.

-   **Phase 1 (Essential for Bootstrap):**
    -   Review Queue Interface
    -   System Health Dashboard
    -   CLI for status checks

-   **Phase 2 (Production & Usability):**
    -   Timeline Viewer
    -   Web-based Configuration Tool
    -   Grid Visualization Tool (for debugging)

-   **Phase 3 (Refinement):**
    -   Face Library Management Interface
    -   Advanced analytics and reporting features

---

## Review Panel Interface

**Purpose:** Efficiently answer "Does this track belong to this candidate?"

**Features:**
-   **Visual Comparison:** Side-by-side view of `Uncertain Track` vs `Candidate` (best 6 frames).
-   **Evidence Summary:** Narrative explanation of system logic (e.g., "Timing plausible," "Face match low").
-   **Actions:** `Merge`, `Reject`, `Create New Agent`, `Skip`.

**LLM Interface:**
-   LLMs receive *only* the visual comparison panels.
-   LLMs are blind to logical/temporal metadata to prevent hallucination.

---

## Timeline Viewer

**Purpose:** Reconstruct agent journeys and surface uncertainty.

**Features:**
-   **Narrative View:** Chronological list of events for a specific `GlobalAgent`.
-   **Uncertainty Markers:** Clearly flag events with low confidence.
-   **Drill-down:** Click any event to view the source video clip.
-   **Filtering:** Filter by date, time, camera, or confidence level.

---

## Health Dashboard

**Purpose:** Real-time system status and long-term learning metrics.

**Key Metrics:**
-   **Processing Lag:** Time from real-world event to digital record.
-   **Auto-Merge Rate:** % of decisions made without human help (Learning Curve).
-   **Queue Depths:** Review Queue and Merge Queue sizes.
-   **Alerts:** Proactive warnings for saturation or failures.
