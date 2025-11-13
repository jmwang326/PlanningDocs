# Master List — Hard vs Figure-Out

This is a living list to keep us aligned on what's genuinely hard (deep/tunable) versus what's a standalone, solvable task we just need to figure out and check off.

**HARD** = Requires tuning, calibration, or judgment calls based on real data  
**FIGURE-OUT** = Deterministic implementation (works or doesn't, may need debugging)

## Quick defaults (reference)
- grid_intra: 32×32 (perspective-aware, per camera)
- grid_inter: 8×8 (clean 4:1 downsampling from 32×32)
- discovery_window_s: 40
- merge_auto_threshold: 0.80 (auto-merge if score ≥ 0.80)
- merge_suggest_threshold: 0.50 (queue for LLM/manual if 0.50-0.80)
- face_match_threshold: 0.70 (CodeProject.AI face recognition auto-merge)
- llm_yes_min: 0.80; llm_no_min: 0.70
- edge_learning_alpha: 0.3-0.5 (fast learning on validated merges)
- overlap_detection_window: 0.5s (Δt < 0.5s = potential overlap)
- panel_size: 768×768 (1 Gemini Flash 2.0 tile)
- panel_layout: 2 rows × 6 images (12 total)
- crop_padding: 5% per side
- bi_timeout_ms: 5000; image_retry_backoff_s: [0.25, 0.5, 1.0, 2.0]
- frame_retention_days: 9; metadata_retention: indefinite
- grid_backup_frequency: weekly (versioned snapshots of learned edges, height calibration)
- lpr_enabled: per-camera checkbox (optional, expensive)

## Hard problems (deep/tunable)

### **Configuration Stage Tuning (Resource Throttling)** [Phase 0]
- Why: DEV_SLOW needs to prove logic on underpowered hardware without melting CPU; PROD_AUTO needs full throughput.
- De-risk: Start DEV_SLOW (2 FPS, 5 cameras, no merging); measure CPU/memory baseline; gradually increase frame rate until CPU hits 80%; lock those limits per stage.

### **External AI Service Health Thresholds** [Phase 2]
- Why: When to circuit-break failing CodeProject.AI service? When to prefer slower service over faster-but-overloaded?
- De-risk: Track response times (p50/p95/p99), queue depth, success rate per service; circuit-break after 5 consecutive errors; prefer lower-priority service if higher-priority queue depth >10 or p95 latency >500ms.

### **Agent Merge Confidence Thresholds** [Phase 2]
- Why: Auto-merge at 0.80, suggest at 0.50-0.80 — will these work in practice?
- De-risk: Start conservative (0.85 auto, 0.60 suggest); log all decisions for 1 week; analyze false positive/negative rates; adjust thresholds based on accuracy vs. manual review volume trade-off.

### **Height Calibration Convergence (32×32 per camera)** [Phase 1]
- Why: How many observations per cell needed before trusting depth estimates? Affects movement calculations.
- De-risk: Track variance per cell; flag as "calibrated" after 20+ observations with stddev < threshold; use uniform depth assumption until calibrated; visualize calibration heatmaps per camera.

### **32×32 Perspective Parameters per Camera** [Phase 1]
- Why: `k_row`, `horiz_vanish_strength`, etc. affect depth/distance calculations. Manual tuning is burdensome.
- De-risk: Start uniform (no perspective); enable perspective only after height calibration proves stable; consider auto-calibration from height gradient if manual tuning too painful.

### **Portal Time Window Tuning (Manual Configuration)** [Phase 2]
- Why: How long should we wait for continuation through portal? User-configured, can optionally learn from validated merges.
- Decision: **Manual YAML config** - user sets typical_time, max_time per portal; can learn tighter windows from validated merges in Phase 4.
- De-risk: Start with conservative max_time (10-20s); tighten after observing validated merge timing distributions.

### **Portal Discovery (Optional, Phase 3+)** [Phase 3]
- Why: Finding portals (doors/gates) from observations could suggest configurations.
- De-risk: On validated merge with Δt < 5s at consistent cells, suggest portal to user; user approves/rejects; system never auto-creates portals without approval.

### **Multi-Person Crosses and ID-Flip Prevention (32×32 Intra-Camera)** [Phase 1]
- Why: Close passes cause agent ID swaps; YOLO bbox can jump between people.
- De-risk: Spatial continuity checks (can't jump >5 cells); motion vector consistency (direction change >90° suspect); flag `possibly_switched` when uncertain; review panels show pre/post crossover frames.

### **Vehicle Association Scoring Weights** [Phase 3]
- Why: Recency vs. proximity vs. face-in-windshield — which matters most for "who got in the car?"
- De-risk: Start equal weights; log all vehicle departure scenarios; analyze corrections; adjust weights based on which evidence most reliable in practice.

### **Panel Image Selection Quality** [Phase 2]
- Why: Confidence-based selection might not give best visual evidence for LLM (could pick blurry high-confidence over sharp low-confidence).
- De-risk: Start with confidence only (simple); log LLM decision confidence vs. detection confidence; if weak correlation, add sharpness/size/brightness scoring; **avoid complex metrics like Laplacian variance** (hard to codify, tune, and debug); prefer simple heuristics (bbox area, brightness mean/std, detection confidence).

### **Face Recognition Thresholds (CodeProject.AI)** [Phase 2]
- Why: CodeProject.AI face recognition needs match thresholds (e.g., ≥0.70 same person, ≤0.40 different).
- De-risk: Start with CodeProject.AI defaults; log all face match scores vs validated merges; calibrate thresholds per camera if needed (some angles/lighting more reliable); unknown faces get added to library on first validated merge.

### **Face Quality Gating** [Phase 2]
- Why: Need minimum quality threshold to avoid registering/matching bad face crops.
- De-risk: Simple quality = bbox_area × detection_confidence × brightness_ok; CodeProject.AI already does face detection; if face_confidence < 0.5, skip; no complex blur metrics initially; add only if needed based on false positive rate.

### **Night/Rain/IR Robustness** [Phase 1-2]
- Why: Noise, blur, and artifacts differ dramatically day vs. night; detection confidence and quality thresholds need separate profiles.
- De-risk: Day/night flag per camera (time-based or lux sensor); separate confidence thresholds; IR-aware quality scoring; stricter gates at night; log per-condition accuracy.

### **LLM Adjudication Reliability and Schema Discipline** [Phase 2]
- Why: LLM may drift from JSON schema, give uncertain answers, or be influenced by prompt variations.
- De-risk: Strict JSON schema validation; temperature=0; tight crops (no background distractions); cache responses to avoid re-asking; test with known same/different pairs; local validator rejects malformed responses.

### **Portal Confidence Decay (Stale Portals)** [Phase 4]
- Why: Portals not seen recently should be questioned (camera moved, door blocked, etc.).
- De-risk: Time-based decay (e.g., confidence -= 0.01 per week without observation); portals below min confidence (e.g., 0.30) marked provisional; re-promotion requires new validated observations.

### **Re-ID Threshold Calibration** [Phase 4-5, Deferred]
- Why: Per-camera Re-ID fingerprints need similarity thresholds (e.g., ≥0.85 auto-merge, ≥0.70 suggest).
- De-risk: Defer until Phase 4-5; face recognition (CodeProject.AI) is primary automation path; Re-ID useful for vehicles and faces that fail face recognition; build database opportunistically from validated merges.

### **Re-ID Model Selection and Performance** [Phase 4-5, Deferred]
- Why: OSNet, FastReID, or custom? Trade-off between accuracy and speed (target <10ms per embedding).
- De-risk: Defer until face recognition proven; Re-ID as fallback/enhancement; benchmark models only after face recognition deployed.

### **Re-ID Fingerprint Diversity per Person** [Phase 4-5, Deferred]
- Why: Clothes get reused; need multiple fingerprints per agent per camera to handle wardrobe rotation (e.g., 3-5 outfits).
- De-risk: On positive ID, compute similarity to existing fingerprints for that agent on that camera; if new fingerprint differs significantly (cosine < 0.80) from all existing, keep it as new variant; if similar to existing (≥0.80), discard (redundant); max 5 fingerprints per agent per camera; prune least-used when adding 6th.

## Figure-out tasks (standalone, solvable)

### **Configuration System (Stages, Hot-Reload, Secrets)** [Phase 0]
- Outcome: YAML config with stages (DEV_SLOW, DEV_FULL, PROD_LEARNING, PROD_AUTO); hot-reload without restart; encrypted secrets (DPAPI/AES); version stamping on artifacts; validation + auto-rollback.
- Test: Edit config file; verify changes applied within 5s; rollback to previous version; test secrets encryption/decryption; verify artifacts stamped with config_version.

### **External AI Service Registry & Scheduler** [Phase 2]
- Outcome: Multi-service support (CodeProject.AI instances); priority routing (try priority 1 first, failover to priority 2); health tracking (response time, queue depth, success rate); circuit breaker (open after 5 errors, cooldown 60s); per-service model checkboxes (face_recognition, lpr).
- Test: Register 2 services (GPU + CPU); route requests to GPU first; simulate GPU failure → verify failover to CPU; verify circuit opens after 5 errors; verify health metrics updated (p50/p95/p99 latency, queue depth).

### **Resource Throttling (Adaptive Frame Rate)** [Phase 0]
- Outcome: Monitor CPU every 10s; if CPU >80% for 30s, reduce frame_rate by 0.5 FPS; if CPU <70% for 60s, increase by 0.25 FPS; respect stage-defined min/max frame_rate.
- Test: Start DEV_SLOW at 2 FPS; simulate high CPU load → verify frame rate reduced; simulate low CPU → verify frame rate increased; verify never exceeds stage max.

### **BI JPEG Acquisition @10 FPS** [Phase 0]
- Outcome: Sustained feed with <0.5% drops; auto-recover on 401/5xx; keep-alive connections; retry with exponential backoff.
- Test: Run for 24 hours, measure drop rate and recovery time.

### **YOLO Detection on Sub Frames** [Phase 0]
- Outcome: YOLOv8 small detects person/vehicle/animal on 960px sub frames; bbox coordinates returned; confidence scores available.
- Test: Process 1000 frames, validate detections visually, measure inference time (<50ms per frame).

### **32×32 Grid Overlay and Cell Mapping** [Phase 1]
- Outcome: Pixel bbox centroid → (row, col) cell in 32×32 grid; perspective-aware spacing (configurable per camera).
- Test: Map 100 detections to cells, visualize on frame, verify larger cells at bottom/center.

### **8×8 → 32×32 Downsampling** [Phase 2]
- Outcome: Cell mapping function `(row32, col32) → (row8, col8)` via `// 4`; all 1024 cells in 32×32 map to exactly one cell in 8×8.
- Test: Verify mapping bijection; test edge cases (0,0) and (31,31).

### **Frame Storage (Ring Buffer + Persistence)** [Phase 0]
- Outcome: R seconds (e.g., 7s) pre-roll ring buffer per camera; flush to disk on Armed/Active; crash-safe manifests.
- Test: Trigger Active, verify pre-roll included; kill process mid-flush, verify recovery.

### **Manifest Generation (Atomic Writes)** [Phase 0]
- Outcome: JSON manifest lists all frames for reconstruction; atomic writes (temp file + rename); reconstructable with ≤1 frame gap.
- Test: Generate manifest for 100-frame sequence; verify all frames listed; corrupt test by killing mid-write, verify recovery.

### **Agent Persistence Across Track Gaps** [Phase 2]
- Outcome: Agent object continues conceptually when track segment ends; track segments linked to agent; agent ID stable across cameras.
- Test: Track ends on CamA, starts on CamB within plausible window; verify same agent ID linked.

### **Multi-State Location Tracking** [Phase 3]
- Outcome: Agent can have `possible_locations[]` with confidence scores; uncertainty collapses on later observation.
- Test: 2 people enter garage, 1 car leaves → both agents show uncertain states; one exits on foot → other's confidence increases.

### **Panel Builder (768×768, 2×6 Layout)** [Phase 2]
- Outcome: Generate panel from two track segments; 6 images per person; confidence-based selection (simple: bbox area × detection confidence); proper letterboxing; saves to disk.
- Test: Build panel from recorded tracks; visual inspection confirms alignment; LLM can parse image.

### **Crop from Main with Bbox Scaling** [Phase 2]
- Outcome: Detect on sub (960px), fetch main (1920px), scale bbox 2x, add 5% padding, crop from high-res.
- Test: Compare crop quality (main vs. sub); verify no coordinate misalignment; measure crop time (<100ms).

### **LLM Integration with Image (Gemini Flash 2.0)** [Phase 2]
- Outcome: Single test call with 768×768 panel returns valid JSON; schema validated; handles same/different person correctly.
- Test: 10 known same-person panels → "same_person": true; 10 known different → false; all return valid JSON.

### **Height Calibration Online Learning** [Phase 1]
- Outcome: Per-camera × cell_32x32 median height updates from observations; cold start assumes uniform; converges after 20+ samples.
- Test: After 100 person detections, cells show depth gradient (taller bottom, shorter top); variance below threshold.

### **Intra-Camera Tracking (IoU Association)** [Phase 1]
- Outcome: Frame-to-frame bbox association via IoU; tracks maintain agent ID; handles brief occlusions (<1s).
- Test: Track single person for 60s with brief tree occlusion; verify single agent ID maintained.

### **Camera State Machine (Agent-Driven)** [Phase 0]
- Outcome: Camera transitions Standby → Armed → Active → Post based on agent activity; inference sampling adjusts per state.
- Test: Quiet → detection → sustained movement → quiet; verify state transitions and inference rate changes.

### **Edge Storage and Retrieval (8×8 Inter-Grid)** [Phase 2]
- Outcome: Edge records (src/dst cameras, cells, min/typical/P90 times, confidence, support) persist to DB; queryable by src camera+cell.
- Test: Store 100 edges; query edges from CamA cell (7,4); verify retrieval time (<10ms).

### **Observable Evidence Scoring** [Phase 2]
- Outcome: Score merge candidates using timing, spatial, face match (if available); return score + evidence list.
- Test: Score 10 known good merges (should be >0.80); 10 bad merges (should be <0.50); verify evidence list explains score.

### **Face Recognition Integration (CodeProject.AI)** [Phase 2]
- Outcome: Send face crops to CodeProject.AI; get back identity match + confidence; build face library incrementally; unknown faces registered on validated merge.
- Test: Register 5 known people; match 20 face crops → ≥0.70 for same person, ≤0.40 for different; timeouts/retries handle service downtime; <200ms per face match.

### **Face Library Management** [Phase 2-3]
- Outcome: Add faces to library on validated merge (human/LLM confirms identity); multiple faces per person across cameras/lighting; prune low-quality faces; track match statistics.
- Test: Register person with 3 good faces; 4th poor quality rejected; 5th different angle added; match rate >80% on known people.

### **Manual Review UI (Queued Decisions)** [Phase 2]
- Outcome: Queue pending merge decisions; display panels side-by-side; accept/reject/defer; log corrections.
- Test: Queue 5 decisions; human reviews; corrections recorded; system learns from feedback.

### **Face Crop Extraction from Tracks** [Phase 2]
- Outcome: Extract face crops from person tracks; simple quality filter (bbox area, detection conf, brightness); save top 3-5 per track; ready for CodeProject.AI face recognition.
- Test: Track person for 10s; extract 5 face crops; all meet quality threshold; crops saved with metadata.

### **CodeProject.AI Wrappers (Face Recognition)** [Phase 2]
- Outcome: Timeouts, retries, rate limits, circuit breaker; >99% success on 100 test crops; trainable face recognition via "register" API.
- Test: 100 face crops → identity returned or "unknown"; register 5 new people; match rate >80%; handle service downtime gracefully.

### **LPR Integration (CodeProject.AI - Optional)** [Phase 5]
- Outcome: LPR on vehicle crops; normalized plate strings; rate limiting; circuit breaker.
- Test: 50 plate crops → normalized strings; handle low-confidence gracefully; fallback when service down.

### **MP4 Reconstruction from Frames** [Phase 0]
- Outcome: Build MP4 from manifest JPEG frames; constant 10 FPS; smooth playback; no audio.
- Test: Reconstruct 30s clip from 300 frames; verify frame count, playback smoothness, duration accuracy.

### **Storage Telemetry Hooks** [Phase 0]
- Outcome: Measure F_sub/F_main average sizes; daily stats; estimate retention needs; alert on disk usage.
- Test: Run for 24h; verify stats logged; retention estimates match reality within 10%.

### **Secrets Storage (BI, LLM API Keys)** [Phase 0]
- Outcome: Secrets encrypted (DPAPI/AES); reveal timer; test & save workflow; no plaintext in config files.
- Test: Store BI credentials; retrieve and use; verify no plaintext in logs or configs.

### **Portal State Backup (Weekly Snapshots)** [Phase 2]
- Outcome: Weekly automated backup of portal configuration, height calibration (32×32), grid parameters; versioned snapshots (timestamp + hash); export to JSON for portability.
- Test: After 2 weeks, verify 2 backup snapshots exist; restore from backup; verify portals/heights match snapshot state.

### **Portal Visualization (Configuration Dashboard)** [Phase 3]
- Outcome: Web dashboard showing configured portals with timing stats; heat maps of height calibration (32×32) per camera; time-series of portal usage, auto-merge rate; validation metrics.
- Test: View dashboard; see portals with timing distributions; heat map shows calibrated vs. uncalibrated cells; track portal usage over time.

### **Camera Grid Viewer Update (32×32 Uniform Grid)** [Phase 0]
- Outcome: Update existing `sandbox_cameraconfig/camera_dashboard_4x5.py` to show 32×32 uniform grid overlay (remove old 12×12 perspective grid); keep ignore zone polygon drawing; toggle grid with `g` key.
- Test: Run viewer; press `g` to toggle 32×32 grid overlay; draw ignore zone polygon; verify saved to `overlay_config.json` with normalized coordinates.

### **CLI Health Monitor** [Phase 0]
- Outcome: Click-based CLI tool with commands for health checks (system, cameras, services, storage), config management (show, validate, reload, rollback), secrets (set, reveal, test), system control (start, stop, restart with stage).
- Test: Run `marengo health status` → see system stats; `marengo config reload` → verify config reloaded within 5s; `marengo secrets set bi_password` → verify encrypted in secrets file.

### **Single-Camera Debug Viewer** [Phase 1]
- Outcome: tkinter app showing one camera with detection bboxes, track IDs, 32×32 grid overlay, cell assignments, track list with stats, height calibration bar chart per cell, camera state indicator (Standby/Armed/Active/Post), playback controls.
- Test: Select camera; see live detections with track IDs; toggle grid overlay; verify cell IDs correct; pause and step through frames; check height calibration chart updates.

### **Merge Decision Queue (Web UI)** [Phase 2]
- Outcome: Flask web app (port 5001) showing pending merge candidates sorted by confidence; side-by-side panel comparison (768×768, 2×6 layout); evidence summary; accept/reject/defer buttons; accept registers face to library and updates portal timing statistics.
- Test: Queue 10 merge candidates; review via web UI; accept 5, reject 3, defer 2; verify face library updated; portal statistics updated; queue count correct.

### **Face Library Manager (Web UI)** [Phase 3]
- Outcome: Flask web app (port 5003) showing gallery of registered people with face count, match stats, last seen; edit person to rename, delete faces, merge duplicates, add face from track.
- Test: View library with 5 registered people; rename "Unknown1" to "John"; delete poor quality face; merge duplicate person; verify CodeProject.AI library updated.

### **Event Timeline Viewer (Web UI)** [Phase 4]
- Outcome: Flask web app (port 5004) showing chronological activity list filtered by date/person/camera/class; play button reconstructs MP4 from JPEGs (10 FPS); multi-camera playback switches at merge points; download MP4.
- Test: Filter timeline to Nov 7; select Person #42; play clip → see 3-camera sequence; verify transitions smooth; download MP4 and verify playback.

### **Re-ID Database per Camera** [Phase 4-5, Deferred]
- Outcome: Extract Re-ID embeddings from validated merge tracks; store per camera × agent; fingerprint structure includes lighting/clothing variants.
- Test: After 10 validated merges, verify 20+ fingerprints stored; retrieve by agent_id + camera_id; embeddings are 512-dim float32.
- Note: **Deferred until face recognition proven**; Re-ID useful for vehicles and face-match failures; build opportunistically.

### **Re-ID Matching Pipeline** [Phase 4-5, Deferred]
- Outcome: Given new track on camera, compute Re-ID embedding; compare against stored fingerprints for plausible agents; return similarity scores.
- Test: Known same-agent tracks → similarity ≥0.85; known different-agent tracks → similarity <0.70; inference time <20ms per candidate.
- Note: **Deferred until face recognition proven**; use as fallback when face match uncertain or no face detected.

### **Re-ID Maintenance (Pruning & Statistics)** [Phase 4-5, Deferred]
- Outcome: Prune fingerprints >90 days old; **keep max 5 diverse fingerprints per agent per camera** (wardrobe rotation); track match_count and false_positive_count; remove unreliable fingerprints (false rate >30%).
- Test: After 100 Re-ID matches with corrections, verify statistics updated; stale fingerprints removed; unreliable ones pruned.
- Note: **Deferred until face recognition proven**.

### **Re-ID Diversity Check (Avoid Redundant Fingerprints)** [Phase 4-5, Deferred]
- Outcome: On positive ID, compute cosine similarity between new embedding and all existing fingerprints for that agent on that camera; if all similarities ≥0.80, discard new embedding (redundant); if any similarity <0.80, keep as new clothing variant; enforce max 5 fingerprints per agent per camera (prune least-used when adding 6th).
- Test: Store 3 fingerprints for Agent A (different outfits); add 4th similar to first (cosine 0.85) → discarded; add 5th dissimilar (cosine 0.65) → kept; add 6th dissimilar → least-used pruned.
- Note: **Deferred until face recognition proven**.

## Data we must store now to unlock later (calibration/learning)

### **Panels and LLM Decisions (Indefinite)**
- All 768×768 panels generated for merge candidates (even if auto-merged without LLM)
- LLM responses: prompt version, model ID, raw JSON, reasoning, confidence
- Human overrides: corrections with context (evidence used, scores)
- Purpose: Audit trail, re-training, threshold calibration, panel quality improvement

### **Merge Decisions with Full Context (Indefinite)**
- Source/target track IDs, cameras, timestamps, Δt
- Observable evidence scores (timing, spatial, face match if available)
- Decision: auto-merged, llm-decided, manually-decided, rejected
- Confidence and reasoning
- Purpose: Learn which evidence types matter most; adjust thresholds; analyze false positives/negatives

### **Face Recognition Results (Indefinite)**
- All face match attempts: crop quality, identity returned (or unknown), confidence score, response time
- False positives/negatives from human corrections
- Face registration events: when/why faces added to library, which validated merge triggered registration
- Purpose: Calibrate face match thresholds; understand failure modes; track face library growth; measure automation rate

### **Edge History (8×8 Inter-Grid, Long-Lived)**
- Per edge: all validated observations (Δt, timestamp, agent ID)
- Running stats: min_time, typical_time, P90, variance, support count
- Confidence history (track changes over time)
- Purpose: Detect edge stability vs. drift; identify when recalibration needed; visualize learned adjacency

### **Height Calibration per Cell (32×32, Indefinite)**
- Per camera × cell: all bbox height observations (with timestamp)
- Running stats: median, variance, sample count, calibrated flag
- Purpose: Visualize depth gradient; detect when calibration complete; recalibrate if camera moves

### **Track Segments with Diagnostics (14 days)**
- Agent ID, camera, start/end times, entry/exit cells (32×32)
- Path: sequence of cells traversed
- Detections: sample at ~1 FPS (not full 10 FPS) with bbox, confidence, cell
- possibly_switched flag and timestamp if triggered
- Purpose: Diagnose ID swaps; understand crossing patterns; improve anti-swap logic

### **Vehicle Departure Events (Indefinite)**
- Vehicle ID, garage/portal, departure time
- People present: list of agent IDs with last-seen locations and times
- Associations made: confidence scores, evidence used (recency, proximity, face-in-windshield)
- Human corrections if overridden
- Purpose: Tune vehicle association weights; understand failure modes

### **Night/Conditions Context (Roll-Up Hourly)**
- Per camera: day/night flag, motion_score summary, IR mode
- Hourly histograms: detection counts, avg confidence, quality scores
- Purpose: Separate day/night profiles; detect weather/lighting issues; adjust thresholds per condition

### **Image Bank per Agent (Indefinite)**
- Top 3 images per agent × camera × lighting condition (day/night)
- Crops: 128×384 from main frames with quality scores
- Purpose: Re-ID fallback when face fails; manual identity assignment; visualization

### **Re-ID Fingerprints per Camera (Indefinite, Phase 3+)**
- Per camera × agent: embeddings (512-dim), lighting/clothing variants, quality scores
- Match statistics: match_count, false_positive_count, last_seen_ts
- Reference crops: sample images for audit/verification
- Purpose: Fast per-camera Re-ID; especially effective for vehicles; useful for people within session (same clothing); build from validated merges
- Schema: `reid_fingerprints(fingerprint_id, agent_id, camera_id, embedding, lighting, clothing_variant, quality, created_ts, last_seen_ts, match_count, false_positive_count)`

### **Grid Geometry and Versions (Indefinite)**
- 32×32 perspective parameters per camera (k_row, vanish strength, etc.)
- 8×8 derived mapping (for audit/verification)
- Version stamps on all artifacts that depend on grid geometry
- Purpose: Support recalibration if camera moves; ensure artifacts interpretable after grid changes

This minimal data footprint ensures we can recalibrate thresholds, re-run adjudications, and refine timing without re-capturing frames.

## Midstream flexibility plan (changing policy without losing progress)
- Versioned configs & stamps
  - Stamp manifests, tracks, faces, candidates, and edges with `config_version` and `grid_version`; store AOI `map_version` and prompt `template_version`.
- Hot-reload and feature flags
  - Allow runtime updates for thresholds (Q_min, θ_face, windows) and toggles (LLM enabled, embedding model) with audit trail.
- Backfill jobs
  - Recompute embeddings on stored crops when model/thresholds change; re-score CCCs from cached candidates; re-run LLM on CANNOT cases if policy tightens.
- Idempotent pipelines
  - Use stable IDs for events/tracks/candidates; writes are upserts with version checks; rebuilds don’t duplicate.
- Retention safety window
  - Keep calibration/diagnostic artifacts for ≥14 days during rollout; trim after metrics stabilize.

## Rollout Guidance (High-Level)

See `MASTER_ROLLOUT.md` for complete implementation guide. This section summarizes the philosophy.

### **Phase 0: Foundation**
Prove acquisition and detection work before tracking complexity.
- Figure-out: BI JPEG fetcher, YOLO detection (no motion gate), frame storage, manifests
- Hard: Confidence thresholds per class, day/night robustness
- Success: 10 FPS sustained from all cameras for 24 hours

### **Phase 1: Single-Camera Tracking**
Prove intra-camera tracking before cross-camera merging.
- Figure-out: IoU tracking, 32×32 grid, agent creation, state machine
- Hard: Height calibration convergence, ID-swap sensitivity
- Success: Track person across one camera for 60+ seconds through occlusion

### **Phase 2: Cross-Camera Correlation**
Prove merge logic and portal configuration.
- Figure-out: Portal configuration (YAML), panel builder, LLM integration, **face crop extraction, CodeProject.AI face recognition wrapper, face library management**
- Hard: Portal time windows, **face recognition thresholds, face quality gating**
- Success: Merge person across 2 cameras; 50%+ auto-merge rate after 1 week; **face recognition handles 60%+ of known-person merges**

### **Phase 3: Uncertainty Handling**
Prove multi-state tracking and vehicle containers.
- Figure-out: possible_locations tracking, vehicle association, manual review UI
- Hard: Vehicle association evidence thresholds, uncertainty collapse logic
- Success: Handle "2 people + 1 car" gracefully; resolve via later observations; **face recognition library covers regular visitors**

### **Phase 4: Maturity**
Tighten portal windows, improve confidence.
- Figure-out: Portal window shrinking (learn from validated merges), correction feedback loops, panel pruning, **(optional) Re-ID database building from validated merges**
- Hard: Portal confidence decay tuning, **(optional) Re-ID threshold calibration**
- Success: 90%+ auto-merge rate; tight portal windows; low false merge rate; **face recognition + time/space handle most cases; Re-ID as fallback for vehicles/face-match failures**

### **Phase 5: Nice-to-Haves**
Polish and advanced features (only if needed).
- Figure-out: Identity clustering, **LPR (CodeProject.AI)**, **(optional) Re-ID matching pipeline**, portal/structure semantics
- Hard: Clustering thresholds, portal transit times, **(optional) Re-ID model selection**
- Success: Regular visitors auto-identified; complex scenarios handled; **LPR for vehicle tracking**

### **Why This Ordering**
- Data capture first (can't learn without frames)
- Prove one thing at a time (intra before inter)
- Simple heuristics before complex (confidence before re-ID)
- **Configuration over learning** (portals in YAML, not statistical discovery)
- **No motion gate** (detector confidence + persistence IS the filter)
- **Face recognition before Re-ID** (CodeProject.AI already set up, trainable, more mature)
- **Face recognition takes us out of the loop faster** (auto-merge known people without LLM)
- Defer nice-to-haves until core proven (portal discovery in Phase 3+, Re-ID as fallback in Phase 4-5)
- Learn timing from validated merges (trust human/LLM approvals)

## Notes
- **Configuration over learning:** User knows where doors are; manual YAML beats statistical discovery; learning is for timing refinement only.
- **No motion gate:** Running YOLO at fixed Hz anyway; detector confidence + persistence (≥3 frames) = real agent; motion gate is unnecessary complexity.
- **Portal-based discovery:** Overlaps (Δt ≈ 0) through portals enable instant auto-merge without LLM.
- **Observable evidence only:** No statistical priors or routine assumptions; only what's visible in footage.
- **Face recognition first:** CodeProject.AI already available; trainable face recognition gets us out of the loop fastest; handles known people automatically.
- **Re-ID as fallback/enhancement:** Deferred to Phase 4-5; useful for vehicles and when face match fails/uncertain; build opportunistically from validated merges.
- **Portal configuration Phase 2, discovery suggestions Phase 3+:** User marks doors in YAML; optional discovery mode suggests new portals for approval.
- **Panel generation for all candidates:** Even auto-merged cases get panels (audit trail).
- **Simple quality metrics:** bbox area × detection confidence; avoid Laplacian and complex CV metrics (hard to tune/debug).
- **This list is living:** Add/remove items as we validate on real footage; priorities may shift based on what breaks first.
