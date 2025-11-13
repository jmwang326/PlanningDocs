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
- Standby: sparse sampling (~10% of frames)
- Armed: medium sampling (~60%)
- Active: all frames (100%)
- Post: medium sampling

**Why adaptive:** GPU budget limited, prioritize Active cameras

## Tracking (Intra-Camera)

### Track Segments
**YOLO11 provides tracking** - links detections across frames
- Track ID assigned per camera
- Bounding box + confidence per frame
- Track segment = continuous observation on one camera

### Track Persistence
**When does track start/end:**
- Start: first detection above confidence threshold
- Continue: YOLO11 maintains ID across frames
- End: object leaves frame or tracking lost

### Track Quality
**Filters for "real" activity:**
- Duration: track must persist (e.g., ≥3 seconds)
- Movement: sustained motion (not static)
- Confidence: detector confidence above threshold

## Camera States

### State Machine
**Cameras transition based on detected activity:**

**Standby → Armed:**
- Trigger: detection appears
- Behavior: increase inference rate, begin saving frames

**Armed → Active:**
- Trigger: track sustained (duration threshold met)
- Behavior: 100% inference, save all frames (sub + main)

**Active → Post:**
- Trigger: no detections for quiet period
- Behavior: continue saving through post-roll

**Post → Standby:**
- Trigger: post-roll period expires
- Behavior: return to sparse sampling, ring buffer only

### Neighbor Boost
**Adjacent cameras elevate when neighbor Active:**
- Camera A goes Active → Cameras B, C promote to Armed
- Adjacency defined by portal configuration
- Increases chance of catching agent transition

## Related Documents
- **L2 (Strategy):** Why this approach filters noise
- **L11_Detection:** Technical specs (YOLO model, batch processing, GPU scheduling)
