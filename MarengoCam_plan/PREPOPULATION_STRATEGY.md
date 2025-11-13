# BVR Prepopulation Strategy

## Overview
Use Blue Iris historical BVR recordings to pre-build baseline databases before going live. This establishes known people, validates grid learning, and avoids GPU contention during live acquisition.

## Status: Ready to Execute (Waiting on New GPU)

## What We Have

### Working Tools
- **`dev_tools/bvr_replay/gpu_benchmark.py`** - Proven BVR extraction framework
  - Extracts frames from BVR files (116 FPS decode)
  - Handles corrupt headers with `-err_detect ignore_err`
  - Processes 700 frames in ~6 seconds
  - Clean temp file management
  - Saved learned grid to `learned_grid_D5442IP50.pkl`

### Available Data (After Camera Rename)
- **Grid Training:** Limited (cameras just remapped, old BVR data not useful)
- **Face/ReID Training:** Abundant (people are people regardless of camera name)
- **Object Detection:** Working (truck 60%, car 37% validated)

## Prepopulation Plan

### Phase 1: Build Person Databases (HIGH PRIORITY)
**Purpose:** Establish baseline of known people before going live

**Why Pre-build:**
- Current GPU: 82ms inference = can't handle live + embedding extraction
- New GPU: Fast enough for live, but baseline still valuable
- Manual labeling easier offline (no live system disruption)
- Quality control before committing to database

**Process:**
```
BVR Files → Extract Frames → Detect Persons → Crop → Embeddings → Cluster → Label
```

**Tool:** `dev_tools/bvr_replay/build_person_dbs.py` (created, not yet run)

**Outputs:**
- `facedb_{camera}.json` - Face embeddings + metadata
- `reiddb_{camera}.json` - ReID embeddings + metadata
- `person_crops/` - All crops for manual review

**Estimated Processing:**
- 7 BVR files × 100 frames each = 700 frames
- ~1-3% person detection rate = 7-20 person crops
- Processing time: ~10-15 minutes on current GPU

**Next Steps:**
1. Run `build_person_dbs.py` on all available BVR files
2. Review crops in `person_crops/` directory
3. Create clustering tool (DBSCAN/HDBSCAN)
4. Manually label clusters (family members, frequent visitors)
5. Build FAISS index for fast lookup (<10ms)

### Phase 2: Grid Learning (LOWER PRIORITY)
**Purpose:** Pre-populate 32×32 grid with baseline learned times

**Limitation:** Camera names just changed, old BVR data maps to wrong camera names

**Options:**
1. **Wait for live data** - Preferred, grid learns quickly (8.1% in 238 frames)
2. **Manual grid mapping** - Map old camera names to new, reprocess BVR
3. **Bootstrap from defaults** - Start with 3-30s defaults, learn from live

**Recommendation:** Skip BVR grid prepopulation, learn from live
- Grid learns fast (1.21s min learned vs 3s baseline in first session)
- Spatial propagation accelerates learning
- BVR camera name mismatch adds complexity
- Live data is more accurate (current camera positions)

### Phase 3: Validation Runs (BEFORE COMMITTING)
**Purpose:** Validate prepopulated databases before going live

**Face Database Validation:**
- Load FAISS index at startup
- Test search speed (<10ms required)
- Verify cluster quality (visual inspection)
- Test unknown handling (should queue for review)

**ReID Database Validation:**
- Same person across different appearances
- Clustering quality (clothing changes)
- False match rate (different people, same clothes)

## Implementation Workflow

### Week 1: Database Building (Offline)
```bash
# 1. Run person database builder
python dev_tools/bvr_replay/build_person_dbs.py

# 2. Review crops
explorer dev_tools/bvr_replay/output/person_crops

# 3. Create clustering tool (NEW - not yet created)
python dev_tools/clustering/cluster_embeddings.py

# 4. Manual labeling (NEW - not yet created)
python dev_tools/clustering/label_ui.py

# 5. Build FAISS index (NEW - not yet created)
python dev_tools/clustering/build_index.py
```

### Week 2: Validation & Tuning
- Load index at startup
- Test search performance
- Adjust clustering parameters (eps, min_samples)
- Re-label if needed

### Week 3: Live Integration
- Frame loop loads baseline databases
- Match new detections against FAISS index
- Queue unknowns for batch processing
- Grid learns from live data (no BVR needed)

## Tools Needed (Not Yet Created)

### 1. Clustering Tool
```python
# dev_tools/clustering/cluster_embeddings.py
# - Load embeddings from facedb/reiddb JSON
# - DBSCAN/HDBSCAN clustering
# - Parameter tuning (eps, min_samples)
# - Visualization of clusters
```

### 2. Labeling UI
```python
# dev_tools/clustering/label_ui.py
# - Show cluster centroids
# - Manual labeling (assign names/IDs)
# - Unknown handling (leave unlabeled)
# - Export labeled database
```

### 3. FAISS Index Builder
```python
# dev_tools/clustering/build_index.py
# - Build index from labeled embeddings
# - Benchmark search speed
# - Save index to disk
# - Validate accuracy on known samples
```

## Blue Iris Integration

### Current Setup
- **Server:** 192.168.1.100:5555
- **Credentials:** ASST/ASST
- **API:** `/admin?camera={shortname}&user={user}&pw={hash}`
- **BVR Location:** `C:/BlueIris/New/*.bvr`

### BVR File Discovery
```python
# Pattern from gpu_benchmark.py:
bvr_dir = Path("C:/BlueIris/New")
bvr_files = sorted(bvr_dir.glob("*.bvr"))

# Filter by camera (if needed):
camera_pattern = f"*{camera_shortname}*.bvr"
```

### Frame Extraction (Proven Pattern)
```python
# From gpu_benchmark.py (116 FPS decode):
cmd = [
    "ffmpeg", "-err_detect", "ignore_err",
    "-i", str(bvr_file),
    "-vframes", str(max_frames),
    "-f", "image2pipe",
    "-vcodec", "mjpeg",
    "pipe:1"
]

# Parse frames, sample every Nth, clean temps
```

## Camera Name Mapping Issue

### Problem
BVR files created before camera rename:
- Old names: D5442IP48, D5442IP49, D5442IP50
- New names: FrontDoor, Backyard, Driveway (example)
- BVR filenames may contain old shortnames

### Impact on Prepopulation
- **Grid Learning:** ❌ Old BVR → wrong camera → spatial model invalid
- **Face/ReID:** ✅ Person identity independent of camera name

### Solution
**For Person Databases:**
- Process all BVR files regardless of camera name
- Merge person crops from all cameras
- Cluster across entire dataset
- Label is person-centric, not camera-centric

**For Grid Learning:**
- Skip BVR prepopulation
- Bootstrap with defaults (3-30s per cell)
- Learn from live acquisition (correct camera positions)

## Performance Projections

### Current GPU (82ms inference)
- **BVR Processing:** 116 FPS decode + 12 FPS inference = sustainable
- **Live Acquisition:** 10 FPS × 3 cameras = 30 FPS (bottlenecked if all active)
- **Recommendation:** Pre-build databases offline to avoid contention

### New GPU (Coming Soon)
- **BVR Processing:** Faster batches (5-10× speedup likely)
- **Live Acquisition:** 10 FPS × 10+ cameras (no bottleneck)
- **Recommendation:** Still pre-build baseline, then augment from live

## Data Retention & Cleanup

### BVR Files
- Blue Iris retains ~7 days (configurable)
- Process before rotation if possible
- Coordinate with BI retention policy

### Generated Crops
- Keep person crops indefinitely (training data)
- Face crops: top-K per track (disk space management)
- Raw frames: Delete after processing (no value)

## Success Criteria

### Minimum Viable Baseline
- ✅ At least 50 person crops across all cameras
- ✅ At least 5 unique face clusters
- ✅ FAISS search <10ms
- ✅ Manual labels for family members

### Ideal Baseline
- 🎯 200+ person crops (diverse appearances)
- 🎯 10+ unique face clusters
- 🎯 High-quality face crops (blur/pose filtering)
- 🎯 ReID clusters validate across clothing changes

## Next Session Checklist

**When you return:**
1. ✅ Phase 2 Event Inference complete (state machine working)
2. ⏳ New GPU validated (performance benchmarks)
3. ⏳ Person database builder run (`build_person_dbs.py`)
4. ⏳ Clustering tools created (DBSCAN, labeling UI, FAISS)
5. ⏳ Baseline databases loaded into frame loop
6. ⏳ Live acquisition test with database lookups

**Priority Order:**
1. NEW GPU: Benchmark performance
2. RUN: `build_person_dbs.py` on all BVR files
3. CREATE: Clustering tools (DBSCAN → Label → FAISS)
4. VALIDATE: Search speed and accuracy
5. INTEGRATE: Load databases into frame loop
6. TEST: Live acquisition with lookups

**Skip for Now:**
- ❌ BVR grid prepopulation (camera name mismatch)
- ❌ Historical event reconstruction (learn from live)

## Files Reference

### Completed
- `dev_tools/bvr_replay/gpu_benchmark.py` - Extraction framework ✅
- `dev_tools/bvr_replay/build_person_dbs.py` - Database builder ✅
- `src/events/camera_state.py` - Event state machine ✅
- `docs/PHASE2_EVENT_INFERENCE_COMPLETE.md` - Phase 2 summary ✅

### Pending
- `dev_tools/clustering/cluster_embeddings.py` - NOT CREATED
- `dev_tools/clustering/label_ui.py` - NOT CREATED
- `dev_tools/clustering/build_index.py` - NOT CREATED

### Configuration
- `config/cameras.yaml` - Camera definitions (new names)
- `config/secrets.yaml` - BI credentials, CPAI endpoint
- Blue Iris BVR location: `C:/BlueIris/New/*.bvr`

---

**Start here when GPU arrives:** Run `build_person_dbs.py`, then create clustering tools.
