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
- Armed cameras: sub only
- Standby cameras: sub only

## Post-roll

**Continue acquisition after activity ends:**
- Camera degrades from Active → Post
- Continue saving frames through post-roll period (10-15 seconds)
- Captures agent exit/departure

**Note:** No pre-roll buffer needed. Detection is fast, property is large. Missing initial 2-3 seconds of approach is acceptable.

## Storage Strategy

### Frame Persistence
**When to save frames:**
- Standby: not persisted (only kept in 25s processing buffer)
- Armed/Active/Post: all frames saved to disk
- Manifest tracks frame metadata for reconstruction

### Retention
**Frames:** Short-term (days to weeks)
**Metadata:** Long-term (indefinite)
- Frame files pruned after retention period
- Metadata (tracks, merges, agents) preserved

## Frame Buffering & Processing

### 25-Second Frame Buffer
**In-memory buffer for processing:**
- Maintains rolling 25-second window of acquired JPEG frames per camera
- Provides resilience for processing pipeline fluctuations
- Allows YOLO tracker to process 12-second chunks with 2-second overlap

**Why 25 seconds:**
- 12-second chunk duration (with 2s overlap included)
- Chunks extracted every 10 seconds (12s - 2s overlap = 10s stride)
- Need current chunk (12s) + next chunk overlap (2s) + processing headroom (~11s) ≈ 25s total
- Handles processing delays without dropping frames

### Chunk Processing Model
**Processing occurs in overlapping chunks:**
- Every 10 seconds, extract 12-second chunk from buffer
- 2-second overlap between consecutive chunks (chunk 1: 0-12s, chunk 2: 10-22s, chunk 3: 20-32s)
- Overlap allows YOLO tracker to link track IDs across chunk boundaries

**Output:**
- Each chunk packaged with metadata (camera ID, start timestamp, end timestamp)
- Queued for Detection & Intra-Camera Tracking component

## Related Documents
- **L2 (Strategy):** Why we acquire frames this way
- **L3_DetectionAndTracking:** How chunks are processed by YOLO
- **L11_Acquisition:** Technical specs (APIs, retry logic, HTTP transport)
