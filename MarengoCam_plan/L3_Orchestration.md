# Level 3 - Orchestration

## Purpose
To manage the end-to-end evidence processing pipeline, coordinating subsystems to convert raw video chunks into resolved agent identities and timelines.

## High-Level Workflow
1.  **Ingest:** Receive a new 12-second video chunk.
2.  **Process:** Send the chunk to the `ChunkProcessor` to generate `LocalTracks`.
3.  **Link:** Pass new and old tracks to the `TemporalLinker` to connect them across chunk boundaries.
4.  **Resolve:** Send linked tracks to the `IdentityResolver` to assign a `GlobalAgent` ID.
5.  **Persist:** Ensure all track updates are saved to the database.

## Key Considerations
The Orchestrator must handle component failures (e.g., a crash in the `ChunkProcessor`), manage retries, and flag data that requires manual review (e.g., when the `IdentityResolver` is uncertain).
