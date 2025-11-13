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
