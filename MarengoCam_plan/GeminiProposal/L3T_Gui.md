# L3T: GUI Technical Details

This document provides the technical API contracts for the GUI layer. It defines the data exchange format, not the visual implementation.

---

## API Architecture

The system exposes a RESTful API to be consumed by any frontend (Web, CLI, or Mobile).

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

## Data Payloads

### Review Panel Payload (`GET /api/queue/{track_id}/panel`)

```json
{
  "target_track": {
    "id": 101,
    "camera": "Driveway",
    "timestamp": "2023-10-27T14:30:00Z",
    "crop_urls": ["/img/101_1.jpg", "/img/101_2.jpg"]
  },
  "candidates": [
    {
      "agent_id": 55,
      "reason": "Grid Match (0.4s)",
      "confidence": 0.65,
      "crop_urls": ["/img/agent55_best.jpg"]
    }
  ]
}
```

### Review Decision Payload (`POST /api/queue/{track_id}/review`)

```json
{
  "decision": "merge", // or "reject", "new_agent"
  "target_agent_id": 55, // Required if decision is "merge"
  "reviewer_id": "user_jmwang"
}
```
