# L3: Temporal Linking

## Component: `TemporalLinker`

## Purpose

The `TemporalLinker` is the second component in the processing pipeline. Its purpose is to solve the "YOLO ID discontinuity" problem by creating a continuous, unbroken history of an agent's movement **on a single camera**.

While the `ChunkProcessor` creates `LocalTrack`s within a 12-second window, the `TemporalLinker` connects these tracks across consecutive chunks.

## The Problem

The underlying YOLO tracker assigns new tracking IDs whenever a new video chunk is processed. This means the same person visible at `T=11s` in `Chunk_N` and `T=12s` in `Chunk_N+1` will have two different `local_id`s.

## High-Level Process

1.  **Identify Overlap:** The `TemporalLinker` examines `LocalTrack`s from the new chunk (`Chunk_N+1`) and the previous chunk (`Chunk_N`), focusing on the 2-second temporal overlap between them.

2.  **Match Tracks:** For each track in the new chunk, it searches for a corresponding track in the previous chunk based on spatial proximity and appearance similarity within the overlap zone.

3.  **Link and Propagate:** When a match is found, the `previous_track` and `next_track` pointers of the respective `LocalTrack` objects are updated. Any identity information (`global_agent_id` or `identity_candidates`) is also propagated forward to the new track.

## Output

The result is a set of `LocalTrack`s that are now part of a doubly-linked list, forming a continuous chain of observations for each agent on a specific camera. This data is then passed to the `IdentityResolver`.
