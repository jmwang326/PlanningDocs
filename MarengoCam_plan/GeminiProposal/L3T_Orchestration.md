# L3T: Orchestration Technical Details

This document provides the specific technical details for the system's event-driven orchestration.

---

## Data Flow & Trigger Summary

```
[Frame Buffer] (Continuous 10 FPS pull per camera)
       |
       v
[1. Camera State Machine] (Monitors motion, triggers pipeline)
   - Event: Camera transitions from `Active` -> `Post` state.
       |
       v
[2. ChunkProcessor] (Processes the 12s chunk leading up to the event)
       |
       v
[3. TemporalLinker] (Links new tracks to previous chunk's tracks)
       |
       v
[4. IdentityResolver] (Assigns `global_agent_id` or `identity_candidates`)
       |
       v
[Database] (Stores the processed `LocalTrack`s)

--------------------------------------------------------------------

[External Evidence] (e.g., Face match, Human review)
       |
       v
[5. EvidenceProcessor] (Resolves uncertainty for a specific track)
       |
       v
    (Cascade) -> [Propagation & Process of Elimination]
       |
       v
    (Feedback) -> [4. IdentityResolver] (Re-attempts merges with new info)
```

---

## 1. Camera State Machine & Pipeline Trigger

The `ChunkProcessor` is not on a timer. It is triggered by the camera's state machine, which is managed internally by the `ChunkProcessor` for each camera.

-   **`Standby` -> `Armed`**: Motion is detected.
-   **`Armed` -> `Active`**: Motion is sustained for ≥ 3 seconds, promoting a candidate track to a `LocalTrack`.
-   **`Active` -> `Post`**: All `LocalTrack`s on the camera have ended. **This transition triggers the chunk processing pipeline.** A 12-second chunk of frames *leading up to this moment* is sent to the `ChunkProcessor`.
-   **`Post` -> `Standby`**: A configured cool-down period (`post_roll`) expires with no new activity.

This ensures that processing power is only used when meaningful activity has occurred.

---

## 5. The Evidence Cascade

The `EvidenceProcessor` triggers a feedback loop to maximize the impact of new information.

1.  **Receive Evidence:** An event (e.g., human review) resolves `Track_A` to be `Agent_123`.
2.  **Apply & Propagate:** The `global_agent_id` is set on `Track_A` and propagated to all directly linked tracks (via `next_track`, `merged_tracks`, etc.).
3.  **Eliminate Alternatives:** The processor then finds all *other* contemporaneous, uncertain tracks that had `Agent_123` as a candidate and removes it from their `identity_candidates` set.
4.  **Trigger Re-resolution:** If the process of elimination reduces any track's candidate set to a single agent, that track is now also resolved, potentially starting a new cascade.
5.  **Feedback to `IdentityResolver`:** Tracks whose candidate sets were reduced but not fully resolved are sent back to the `IdentityResolver` to see if a merge is now possible with the reduced ambiguity.

---

## Concurrency Model

-   **Frame Acquisition:** One thread per camera, continuously pulling frames into a ring buffer.
-   **State Machine:** One lightweight task per camera, monitoring for state transitions.
-   **Chunk Processing Pipeline:** Each triggered pipeline runs as an independent, parallel task.
-   **Evidence Loop:** A single, asynchronous queue processes all incoming evidence serially. This is critical to prevent race conditions where two different pieces of evidence might try to modify the same track's `identity_candidates` simultaneously.
-   **Database:** A connection pool is used, and all writes are performed within ACID transactions to ensure data integrity.

---

## Error Handling & Timeouts

-   **Frame Fetch Failure:** Log a warning and continue. The ring buffer provides a small degree of fault tolerance.
-   **YOLO Inference Failure:** Log a critical error, discard the current chunk, and reset the camera state to `Standby`.
-   **Database Write Failure:** Roll back the transaction, retry the operation once after a short delay. If it fails again, log a critical alert.
-   **External AI Timeout:** Do not block. Treat the request as a failure, mark the track as uncertain, and if appropriate, add it to the review queue.
-   **Stuck State Machine:** A watchdog timer will monitor each camera's state. If a camera remains in a non-`Standby` state for an abnormally long time (e.g., > 5 minutes) without any state transitions, it will be forcibly reset to `Standby`.
