# Level 3 - Acquisition

## Purpose
How and when we acquire frames from cameras.

## Frame Acquisition

### Source
**Blue Iris JPEG API** - camera connection manager
- Sub profile: lower resolution for detection
- Main profile: higher resolution for detail/face recognition
- No video/MP4 requests (JPEG snapshots only)

### Frame Rate
**Adaptive by camera state:**
- Start: 2-4 FPS (prove system on limited resources)
- Scale to: 10 FPS (face recognition benefits from temporal resolution)
- Per-camera variation based on activity level

### When to Acquire Main Profile
**Sub always acquired, main conditionally:**
- Active cameras: both sub + main
- Armed cameras: sub only (or optional main for neighbors)
- Standby cameras: sub only

## Pre-roll and Post-roll

### Pre-roll Buffer
**Ring buffer in memory** (last 7-10 seconds)
- Always maintained, even during Standby
- Captures lead-up when activity detected
- Committed to disk when camera promotes to Armed/Active

### Post-roll
**Continue acquisition after activity ends:**
- Camera degrades from Active → Post
- Continue saving frames through post-roll period
- Captures agent exit/departure

## Storage Strategy

### Frame Persistence
**When to save frames:**
- Standby: pre-roll only (ring buffer, not persisted)
- Armed/Active/Post: all frames saved to disk
- Manifest tracks frame metadata for reconstruction

### Retention
**Frames:** Short-term (days to weeks)
**Metadata:** Long-term (indefinite)
- Frame files pruned after retention period
- Metadata (tracks, merges, agents) preserved

## Related Documents
- **L2 (Strategy):** Why we acquire frames this way
- **L11_Acquisition:** Technical specs (APIs, retry logic, HTTP transport)

# Level 3 - Video Ingestor

## Purpose
This document defines the tactical plan for the **Video Ingestor**. It describes how video streams are consumed and prepared for processing.

## Tactical Plan

The Video Ingestor is responsible for maintaining a stable, continuous connection to each camera and providing standardized video chunks to the `Detection & Intra-Camera Tracking` component.

### 1. Connection Management
- For each camera defined in the `Configuration System`, the ingestor will establish and maintain a connection to its RTSP stream.
- If a stream connection is lost, the ingestor will attempt to reconnect every 10 seconds, logging the outage.

### 2. The 25-Second Circular Buffer
- For each active camera stream, the ingestor will maintain a **25-second circular buffer** of video frames in memory.
- This buffer provides the resilience needed to handle minor network fluctuations and ensures data is available for chunking.

### 3. Chunk Generation
- The primary output of the ingestor is video "chunks."
- Every **10 seconds**, the ingestor will extract a **12-second chunk** of video from the 25-second buffer.
- This results in a **2-second overlap** between consecutive chunks (e.g., chunk 1 covers 0-12s, chunk 2 covers 10-22s, chunk 3 covers 20-32s, etc.).
- This overlap is critical for the `Detection & Intra-Camera Tracking` component to link tracklets across chunk boundaries.

### 4. Output
- Each 12-second video chunk is packaged with its metadata (camera ID, start timestamp, end timestamp) and placed into a queue for the `Detection & Intra-Camera Tracking` component to consume.
