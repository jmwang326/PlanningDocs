# L3: Acquisition Strategy

**Tactical Goal:** Acquire video frames in a resource-aware manner, increasing fidelity only when necessary to capture meaningful events.

This document outlines the strategy for pulling frames from cameras. The corresponding technical specifications are in `L13_Acquisition.md`.

## Guiding Principles

- **Acquire Just-in-Time:** Frame rates should scale with the level of activity. We do not need to process every frame from every camera all the time.
- **Preserve Pre-Roll:** Always maintain a buffer of recent frames to ensure the beginning of an event is captured.

## Agent-Driven Camera States

To manage resources effectively, each camera will operate in one of four states. The transition between states is driven by the output of the detection and tracking system.

| State | Trigger | Purpose |
|---|---|---|
| **Standby** | No activity detected for a sustained period. | Conserve CPU/GPU resources. A minimal frame rate is maintained to detect new activity. |
| **Armed** | A potential agent is detected with medium confidence. | Increase frame rate to confirm if the detection is a real, persistent agent. |
| **Active** | A track has been established and is being actively followed. | Maximum frame rate to gather high-fidelity data for tracking, merging, and recognition. |
| **Post** | An agent has just left the frame or been lost. | Maintain a high frame rate for a short period to see if the agent reappears or to capture a clean exit for merging. |

The specific frame rates and transition timings for each state are defined in `L13_Acquisition.md`.
