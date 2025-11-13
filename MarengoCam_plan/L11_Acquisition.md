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

## Frame Rate Control

### Adaptive Scheduling
**Per camera, based on state:**
```python
target_fps = {
    'standby': 0.2,   # ~1 frame per 5s
    'armed': 3.0,
    'active': 10.0,
    'post': 5.0
}
```

### Frame Timestamps
**Millisecond precision:**
- Capture timestamp from Blue Iris (if available)
- Or system timestamp at acquisition
- Supports arbitrary FPS (not locked to integer rates)

## Ring Buffer

### Implementation
**Circular buffer in memory:**
- Size: `fps × pre_roll_seconds` (e.g., 10 FPS × 7s = 70 frames)
- JPEG bytes stored (no decode)
- Timestamps + camera_id + frame_number

### Commit to Disk
**Triggered by state transition:**
- Standby → Armed: flush ring buffer to disk
- Continue appending new frames
- Atomic manifest write when complete

## Related Documents
- **Source:** EVENT_INFERENCE_SPEC.md sections (deprecated)
- **L3_Acquisition:** Non-technical how/when
