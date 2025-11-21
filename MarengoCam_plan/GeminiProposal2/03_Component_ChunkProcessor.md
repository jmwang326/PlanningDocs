# 03 - Component: Chunk Processor (The Worker)

## Purpose
The `ChunkProcessor` is the workhorse of the system. It ingests short video segments (chunks), detects objects, tracks them within the camera's view, and extracts evidence.

**Philosophy:**
- **Stateless:** Each chunk is processed independently (mostly). State is passed via the Database or Overlap Buffer.
- **Fail-Safe:** If AI fails, we still record the video and metadata.
- **Resource-Aware:** Respects the VRAM budget defined in `00_Architecture`.

---

## 1. Interface Contract

### 1.1. Primary Function
```python
def process_chunk(
    chunk_path: str, 
    camera_id: str, 
    start_time: float
) -> ProcessingResult:
    """
    Input: Path to a 10-15s video file.
    Output: Status (Success/Fail), Metadata (Processing Time).
    Side Effects: Writes rows to `local_tracks` and `evidence`.
    """
```

### 1.2. Evolutionary Stages (Derisking)

**Stage 1: The Blind Worker (Skeleton)**
- **Logic:** Opens video file, reads duration/FPS, writes a "dummy" track to DB.
- **Goal:** Verify file permissions, DB connection, and pipeline orchestration.
- **No GPU required.**

**Stage 2: The Sighted Worker (YOLO Only)**
- **Logic:** Runs YOLOv8/11. Generates `LocalTracks`.
- **Goal:** Verify object detection and basic tracking (ByteTrack).
- **Output:** Tracks with `class="person"`, but `global_id=NULL`.

**Stage 3: The Intelligent Worker (Full)**
- **Logic:** Adds "Best Shot Selection" and calls CodeProject.AI for Face/Plate recognition.
- **Goal:** Production-ready evidence extraction.

---

## 2. Processing Pipeline

### Step 1: Motion Gate (Optimization)
- **Check:** Fast pixel-difference check on resized frames.
- **Decision:** If motion < threshold, mark chunk as "Empty" and skip YOLO.
- **Benefit:** Saves 80% of GPU cycles at night.

### Step 2: Object Detection (YOLO)
- **Model:** YOLO11n or YOLO11s (Tradeoff: Speed vs Accuracy).
- **Frequency:** Every frame (Active) or Every 3rd frame (Standby).
- **Output:** Bounding Boxes + Class IDs.

### Step 3: Local Tracking (ByteTrack)
- **Logic:** Associates boxes across frames to form a `LocalTrack`.
- **Persistence:** Handles brief occlusions (up to 30 frames).

### Step 4: Evidence Extraction
- **Logic:** Heuristic to find the "Best Shot".
    - Largest Bounding Box.
    - Most Central in frame.
    - Least Motion Blur (Laplacian variance).
- **Action:** Save crop to disk (`/crops/...`).

### Step 5: Biometric Enrichment (Async)
- **Call:** Send "Best Shot" to CodeProject.AI API.
- **Timeout:** **500ms Hard Limit.**
- **Result:**
    - Success: Save Face Embedding to `evidence` table.
    - Fail/Timeout: Log warning, save track without face data.

### Step 6: Commit
- Write `LocalTracks` to DB.
- Write `Evidence` to DB.
- Call `IdentityResolver.resolve_identity(track)` (The Handoff).

---

## 3. Resilience & Error Handling

### 3.1. Corrupt Video Files
- **Symptom:** FFmpeg throws error reading chunk.
- **Action:** Log error, move file to `_quarantine`, skip processing. Do not crash the worker.

### 3.2. AI Service Outage
- **Symptom:** CodeProject.AI returns 500 or times out.
- **Action:** "Circuit Breaker" opens. Skip AI calls for next 30s. Continue processing YOLO tracks (Spatial/Temporal logic still works).

### 3.3. Slow Processing (Lag)
- **Symptom:** Processing 10s of video takes 12s.
- **Action:**
    1.  Enable "Skip Frames" (Process 1 in 2).
    2.  Disable Face Recognition.
    3.  If still lagging, drop chunk entirely (Gap in timeline is better than infinite lag).
