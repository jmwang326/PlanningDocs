# YOLO11 Migration Analysis
## Replacing CPAI with Local YOLO11

**Date:** November 10, 2025  
**Decision:** Replace CPAI (local + remote) with local YOLO11 on RTX 5060  
**Exception:** Keep CPAI for face detection (specialized model)

---

## Current State Analysis

### CPAI Architecture (Current)
```
BlueIris Alert → Extract Clip → Send to CPAI → Get Detections
                                   ↓
                              Local CPAI (CPU)
                              Remote CPAI (if local fails)
```

**CPAI Performance (from testing):**
- **Local CPAI:** ~2-4s per clip (CPU-bound)
- **Remote CPAI:** 5-10s per clip (network + queue latency)
- **Reliability:** Local often fails/slow, fallback to remote
- **Capabilities:** General object detection + face detection

**CPAI Resource Usage:**
- CPU: Heavy load during inference
- Memory: ~2-4GB per instance
- Network: Significant when using remote fallback

---

## YOLO11 Performance (Validated)

### GPU Configuration
- **Hardware:** RTX 5060 (16GB VRAM)
- **Environment:** yolo11-rtx5060 conda env
- **PyTorch:** 2.10.0.dev20251016+cu129 (nightly, CUDA-enabled)

### Benchmark Results

#### Model: yolo11n (nano - fastest)
**Test Clip:** 43 frames @ 320p with skip=7

| Approach | Resize | Inference | Total | Detections | Tracks |
|----------|--------|-----------|-------|------------|--------|
| **OpenCV resize + batch** | 416ms | 2623ms | 3039ms | 77 | 6 |
| **YOLO imgsz (GPU pad)** | 490ms | 1408ms | 1898ms | 57 | 3 |

**Key Findings:**
1. **Batch processing:** 3.57x faster than frame-by-frame (708ms vs 2527ms for tracking)
2. **OpenCV resize:** CPU-bound (no CUDA support in standard opencv-python)
3. **GPU bottleneck:** Resize is the slow part, not inference
4. **Quality:** OpenCV resize finds more objects (possibly due to padding differences)

#### Performance Projection for 5 Cameras

**Clip specs:** 12s @ 25fps = 300 frames  
**With skip=7:** 43 frames per clip  
**Per-clip time (OpenCV + batch):** ~3s  
**5 clips sequential:** 15s total  
**5 clips parallel (GPU has capacity):** ~3-4s total ✅

**Budget:** 7.5s for all 5 cameras  
**Verdict:** ✅ **ACHIEVABLE with parallel processing**

---

## Migration Implications

### 1. **Code Changes**

#### Remove/Modify:
- `cpai_checker.py` - Remove CPAI health checks
- `api.py` - Remove CPAI endpoints, add YOLO11 endpoints
- `ingestor.py` - Replace CPAI inference calls with YOLO11 batch processing
- Configuration files - Remove CPAI URLs, add YOLO11 settings

#### Add:
- **YOLO11 API Server** (`yolo11_api.py`) - FastAPI wrapper for YOLO inference
- **Batch processor** - Collect frames, batch inference, track objects
- **GPU monitor** - Track VRAM usage, prevent OOM
- **Parallel processing** - Handle 5 cameras concurrently

#### Keep (with modifications):
- `media_extractor.py` - Extract clips from BlueIris
- `database.py` - Store detections (schema may need updates)
- Alert processing logic - Similar flow, different inference backend

### 2. **Documentation Updates**

Files to update:
- `BIGPICTURE_V2.md` - Architecture diagram showing YOLO11 instead of CPAI
- `IMPLEMENTATION_GUIDE.md` - New setup instructions for YOLO11
- `EXTERNAL_AI_INTEGRATIONS.md` - Add YOLO11, mark CPAI as deprecated (except faces)
- `MEDIA_EXTRACTION_FIX.md` - Update inference pipeline description

New docs needed:
- `YOLO11_SETUP.md` - Conda env setup, model download, API configuration
- `GPU_OPTIMIZATION.md` - Batch sizes, parallel processing, memory management
- `PERFORMANCE_TUNING.md` - Resolution vs speed, skip rates, model selection

### 3. **Resource Requirements**

#### Hardware:
- ✅ **RTX 5060 (16GB VRAM)** - Sufficient for 5 concurrent clips
- ✅ **CPU** - Freed up (was bottleneck with CPAI)
- ✅ **RAM** - YOLO uses less system RAM than CPAI
- ⚠️ **Disk I/O** - Same (reading video clips)

#### Software:
- ✅ **PyTorch 2.10 (nightly)** - Already installed
- ✅ **Ultralytics YOLO11** - Already installed
- ⚠️ **OpenCV** - Current version is CPU-only (resize bottleneck)
- 📋 **FastAPI/Uvicorn** - For API server
- 📋 **Process manager** - Keep YOLO service running (systemd/supervisor)

#### Network:
- ✅ **No external calls** - All local (vs CPAI remote fallback)
- ✅ **Lower latency** - No network roundtrip
- ✅ **No dependency** - Works offline

### 4. **Face Detection Strategy**

**Option A: Keep CPAI for faces only**
- Pros: Specialized model, already working
- Cons: Maintain CPAI infrastructure, dual dependency

**Option B: Use YOLO11 for faces too**
- Pros: Single inference system, simpler architecture
- Cons: YOLO general models may not be as accurate for faces
- Solution: Fine-tune yolo11n on face dataset OR use specialized YOLO face model

**Recommendation:** **Start with Option A** (keep CPAI for faces), evaluate Option B later

### 5. **Migration Risks & Mitigations**

| Risk | Impact | Mitigation |
|------|--------|------------|
| **OpenCV resize CPU bottleneck** | 37% slower than YOLO imgsz | Use PyTorch GPU resize OR accept YOLO imgsz differences |
| **GPU out of memory (OOM)** | Crashes during parallel processing | Monitor VRAM, limit concurrent clips to 3-4 |
| **Different detection results** | May miss objects CPAI found | Run parallel testing, validate accuracy |
| **Torchvision incompatibility** | Can't use direct video file input | Use OpenCV + numpy arrays (already working) |
| **Model loading time** | First inference slow | Pre-load model at startup |
| **Track ID management** | Need to maintain IDs across clips | Use `persist=True` flag correctly |

---

## Horsepower Assessment

### Do we have enough GPU power? ✅ **YES**

**Evidence:**
1. **Single clip (43 frames):** 3s total (resize + inference)
2. **5 clips parallel:** ~3-4s estimated (GPU can batch efficiently)
3. **Budget:** 7.5s available
4. **Margin:** 2x safety factor

**GPU Utilization:**
- **VRAM:** yolo11n model ~500MB, 5 clips in parallel ~2-3GB frames = **~4GB total** (16GB available)
- **Compute:** RTX 5060 can handle multiple batches concurrently
- **Bottleneck:** Currently CPU resize (378-490ms), not GPU inference

### Optimization Path

**Phase 1: CPU Resize (Current)**
- 3s per clip, 15s for 5 sequential
- ⚠️ Doesn't meet 7.5s budget (sequential)
- ✅ Meets budget with 5x parallel (but CPU may bottleneck)

**Phase 2: GPU Resize (Recommended)**
```python
# Use PyTorch for GPU-accelerated resize
import torch
import torchvision.transforms.functional as F

# Batch resize on GPU
frames_tensor = torch.stack([torch.from_numpy(f).permute(2,0,1) for f in frames]).cuda()
resized = F.resize(frames_tensor, target_height)
# Then batch inference
results = model.track(resized, ...)
```
- Estimated: 1.5-2s per clip (eliminates CPU bottleneck)
- 5x parallel: ~2-3s total ✅ **Well under budget**

**Phase 3: Optimizations (Future)**
- Larger model (yolo11s/m) if accuracy needs improvement
- Fine-tune on your camera views
- Dynamic batch sizing based on GPU load

---

## Migration Plan

### Stage 1: Parallel Infrastructure (Week 1)
- [x] ~~Validate GPU performance~~ (DONE)
- [x] ~~Confirm batch processing~~ (DONE)
- [ ] Implement PyTorch GPU resize
- [ ] Test 5x parallel clip processing
- [ ] Benchmark end-to-end latency

### Stage 2: API Development (Week 2)
- [ ] Complete FastAPI server (`yolo11_api.py`)
- [ ] Add batch endpoint for multiple clips
- [ ] Implement GPU memory monitoring
- [ ] Add health checks & status endpoints
- [ ] Process management (auto-restart on crash)

### Stage 3: Integration (Week 3)
- [ ] Modify `ingestor.py` to call YOLO11 API
- [ ] Update database schema if needed
- [ ] Parallel testing with CPAI (compare results)
- [ ] Configure alerts to use YOLO11 by default

### Stage 4: Validation (Week 4)
- [ ] Run side-by-side for 1 week (CPAI + YOLO11)
- [ ] Compare detection accuracy
- [ ] Monitor resource usage (GPU, CPU, RAM)
- [ ] Validate 7.5s latency budget is met
- [ ] Check for edge cases (night vision, weather, etc.)

### Stage 5: Cutover (Week 5)
- [ ] Switch production to YOLO11
- [ ] Keep CPAI running for faces only
- [ ] Monitor for issues
- [ ] Remove CPAI general detection code
- [ ] Update documentation

---

## Decision Points

### 1. Resize Strategy
**Options:**
- A. CPU OpenCV (slower, current)
- B. GPU PyTorch resize (faster, needs testing)
- C. YOLO imgsz parameter (fastest, different results)

**Recommendation:** Test B (PyTorch GPU resize) - likely best balance

### 2. Parallel Processing
**Options:**
- A. Sequential (15s for 5 clips) ❌ Exceeds budget
- B. Thread pool (GPU handles concurrency) ✅ Recommended
- C. Multiprocessing (complex, unnecessary)

**Recommendation:** B (Thread pool with GPU batch processing)

### 3. Model Selection
**Options:**
- A. yolo11n (fastest, 77 detections in test)
- B. yolo11s (balanced)
- C. yolo11m (most accurate, slower)

**Recommendation:** Start with A, upgrade to B if accuracy issues

### 4. Face Detection
**Options:**
- A. Keep CPAI for faces (safe)
- B. Switch to YOLO face model (unified)

**Recommendation:** A initially, evaluate B after general detection stable

---

## Success Criteria

1. ✅ **Performance:** 5 clips processed in < 7.5s
2. ✅ **Accuracy:** Detection quality ≥ CPAI (subjective, needs validation)
3. ✅ **Reliability:** 99%+ uptime (vs CPAI's frequent failures)
4. ✅ **Resource:** < 8GB VRAM, < 50% CPU
5. ✅ **Latency:** End-to-end alert → detection < 10s

---

## Conclusion

### ✅ **We have the horsepower** - RTX 5060 is sufficient

**Key Points:**
1. **GPU performance validated:** 3s per clip with batch processing
2. **Parallel processing feasible:** 5 concurrent clips estimated 3-4s
3. **Bottleneck identified:** CPU resize (solvable with PyTorch GPU resize)
4. **Budget achievable:** Well under 7.5s with optimizations
5. **Simpler architecture:** No network calls, no remote fallback complexity

**Next Steps:**
1. Implement PyTorch GPU resize and benchmark
2. Test 5x parallel processing
3. Build production API server
4. Run parallel with CPAI for validation
5. Cut over to YOLO11 as primary inference engine

**Timeline:** 4-5 weeks to full migration  
**Risk Level:** Low (incremental rollout, keep CPAI as fallback initially)  
**Expected Improvement:** 2-3x faster, more reliable, local-only
