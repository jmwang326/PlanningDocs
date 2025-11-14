# Level 3 - Detection and Tracking

## Purpose
How we detect objects and track them within cameras.

## Detection

### Detector
**YOLO11** - object detection with built-in tracking
- Classes: person, vehicle, animal
- Confidence threshold filters noise
- Runs on sub profile frames

### When to Run Detection
**Adaptive inference rate by camera state:**
- Standby: sparse (motion gate filters, then minimal YOLO sampling)
- Armed: moderate (increased sampling to confirm sustained track)
- Active: full (maximum YOLO sampling within GPU budget)
- Post: moderate (reduced but still watching for re-entry)

**Motion gate (Standby only):**
- Fast pixel-diff check before YOLO inference
- Filters static frames (no motion = no inference)
- Allows Standby→Armed promotion without full YOLO on every frame
- Only frames with motion delta above threshold proceed to sparse YOLO sampling

**Sampling rate determination:**
- Inference Manager determines actual frame sampling rates based on GPU budget
- Prioritizes Active cameras (e.g., ~3 Hz sustained), allocates remaining to Armed/Post, minimal to Standby
- 10 FPS acquisition ensures smooth video playback, not all frames require inference
- Exact rates are dynamic (L11_InferenceManager implementation detail)

## Tracking (Intra-Camera)

### Chunked Processing
**12-second chunks with 2-second overlap:**
- Process video in 12s segments (T=0-12s, T=10-22s, T=20-32s...)
- First 2s of each chunk overlaps with last 2s of previous chunk
- Overlap allows linking YOLO local IDs across chunks

**Why chunked:**
- Real-time processing: 12s chunks keep up with incoming video
- Multiple streams: pipeline chunks across 10+ cameras
- YOLO ID discontinuity: YOLO track IDs reset each inference session
- Overlap solves: "Local ID 3 in chunk N" = "Local ID 7 in chunk N+1"

### Track Segments
**YOLO11 provides tracking within chunks:**
- Track ID assigned per camera per chunk (local ID)
- Bounding box + confidence per frame
- Track segment = continuous observation within one chunk

**Chunk linking:**
- Agent in overlap zone (T=10-12s) → match across chunk boundary
- Single agent in overlap → deterministic match (position continuity)
- Multiple agents → appearance similarity + spatial continuity
- Build linked list: chunk N track → chunk N+1 track → ...

### Track Persistence
**When does track start/end:**
- Start: first detection above confidence threshold
- Continue: YOLO11 maintains ID across frames within chunk
- End: object leaves frame, tracking lost, or chunk boundary
- Cross-chunk: linked via overlap zone matching

### Track Quality
**Filters for "real" activity:**
- Duration: sustained motion across ≥3 seconds (via candidate track accumulation during Armed)
- Movement: object must be moving (not static)
- Confidence: detector confidence above threshold

**Candidate Track Lifecycle:** Motion gate triggers → Armed state → candidate track accumulates detections → if ≥3s sustained movement → promoted to formal track segment → agent confirmed. If motion stops before threshold → candidate discarded, camera returns to Standby.

## Camera States

### State Machine
**Cameras transition based on detected activity:**

**Standby → Armed:**
- Trigger: detection appears (via motion gate + sparse YOLO sampling)
- Behavior: increase inference rate to moderate, begin saving frames to disk

**Armed → Active:**
- Trigger: candidate track promoted (≥3s sustained movement confirmed)
- Behavior: full inference (all frames), save all frames (sub + main), create formal track segment

**Active → Post:**
- Trigger: no detections for quiet period
- Behavior: continue saving through post-roll

**Post → Standby:**
- Trigger: post-roll period expires
- Behavior: return to sparse sampling, 25s buffer only (no persistence)

**Note:** All cameras polled continuously at 10 FPS. Camera states control inference rate and frame persistence, not acquisition. Each camera independently detects agents via its own motion gate + YOLO sampling.

## Related Documents
- **L2 (Strategy):** Why this approach filters noise
- **L11_Detection:** Technical specs (YOLO model, batch processing, GPU scheduling)
