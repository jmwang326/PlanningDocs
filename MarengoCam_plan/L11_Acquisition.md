# Level 11 - Acquisition Technical Specs

## Purpose
Technical implementation details for frame acquisition.

## Blue Iris API

### Endpoint
```
http://{bi_server}/image/{camera_short_name}?q={quality}&s={size}
```

**Parameters:**
- `q`: JPEG quality (0-100)
- `s`: Size (100 = full, 50 = half, etc.)
- Sub vs Main: determined by size parameter or camera alias

### HTTP Transport
- Keep-alive connections (persistent)
- Retry logic: exponential backoff (1s, 2s, 4s, max 30s)
- Timeout: 5 seconds per request
- Jittered scheduling (prevent thundering herd)

## Frame Acquisition Rate

### Constant Polling
**All cameras polled continuously:**
```python
acquisition_fps = 10.0  # Production (constant)
acquisition_fps_dev = 2.0  # DEV_SLOW mode
```

**Why constant:**
- Ensures smooth video playback (10 FPS baseline)
- Acquisition is cheap (HTTP JPEG fetch)
- Camera states control **inference rate**, not acquisition rate

### Frame Timestamps
**Millisecond precision:**
- Capture timestamp from Blue Iris (if available)
- Or system timestamp at acquisition
- Stored with camera_id + frame_number

## Processing Buffer

### Implementation
**25-second rolling buffer:**
- Size: `10 FPS × 25s = 250 frames` per camera
- JPEG bytes stored (no decode until needed)
- Enables chunked processing: 12s chunks with 2s overlap, extracted every 10s

### Commit to Disk
**Triggered by camera state:**
- Standby: frames kept in buffer only, not persisted
- Armed/Active/Post: frames persisted to disk
- Main profile (high-res) acquired on-demand for Active cameras

## Related Documents
- **L3_VideoIngestor.md:** Frame acquisition tactics
- **L3_DetectionAndTracking.md:** Inference rate control
