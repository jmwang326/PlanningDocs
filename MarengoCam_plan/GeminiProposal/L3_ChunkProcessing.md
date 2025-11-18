# L3: Chunk Processing

## Component: `ChunkProcessor`

## Purpose

The `ChunkProcessor` is the first active component in the data processing pipeline. Its sole responsibility is to convert a raw 12-second video chunk from a single camera into a list of high-quality, filtered `LocalTrack` objects.

## High-Level Process

The conversion is performed in a three-stage filtering process to ensure only significant and sustained activity becomes a `LocalTrack`.

1.  **Motion Gating (for Standby cameras):** A fast pixel-difference check is performed to filter out frames with no significant motion, saving GPU resources by avoiding unnecessary YOLO processing.

2.  **YOLO Detection & Tracking:** The YOLO model is run on frames that pass the motion gate. It detects objects (person, vehicle, animal) and tracks them within the chunk.

3.  **Track Quality Filtering:** Raw YOLO tracks are filtered to remove noise. Only tracks that demonstrate sustained movement (e.g., ≥ 3 seconds) and meet a minimum confidence score are promoted to become official `LocalTrack` objects.

## Output

The final output is a list of `LocalTrack` objects, each representing a significant, continuous observation of an agent within that 12-second chunk. These tracks are then passed to the `TemporalLinker`.
