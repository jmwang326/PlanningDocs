# Event & Inference Spec — Draft

This document details the mechanics behind the high-level plan in `BIGPICTURE_V2.md`. Values marked TBD will be set after initial validation.

## 1. Contracts (Concise)
- Inputs: per-camera JPEG frames at 10 FPS (sub always, main during detail); each frame carries `{camera_id, ts_utc, profile, jpeg_bytes}`.

- Outputs: events with full-fidelity playback windows, manifests for reconstruction, optional MP4 from frames (no audio).
- Success: high precision on "real" activity; smooth 10 FPS playback for Armed→Active→Post windows; safe multi-camera concurrency.
- Error modes: BI throttling, camera offline, GPU saturation, detector failure. Active fidelity must persist through degradations.

## 2. Acquisition
- Fixed 10 FPS per camera (sub profile). Main profile frames only during Active detail windows (neighbors TBD).
- Transport: HTTP keep-alive, short timeouts, jittered scheduling, exponential backoff per camera on repeated errors.
- BI main/sub selection convention: **TBD** (aliases vs width/height parameters). Document once chosen.
- **Grid Systems:** Each camera uses a fixed 32×32 uniform grid (1024 cells) for intra-camera tracking with online height calibration per cell for depth normalization. Each cell is frame_width/32 × frame_height/32 pixels. For inter-camera adjacency learning, we downsample to an 8×8 grid (64 cells) via clean 4:1 mapping (`row_8 = row_32 // 4, col_8 = col_32 // 4`). This dual-grid provides tractability (4K edges per camera pair vs 1M) while maintaining tracking precision. Grid is uniform (not perspective-aware) to remain stable if camera moves.

## 3. Detection (No Motion Gate)
**~~Motion gate removed~~** — originally proposed as pre-filter before YOLO to save GPU budget.

**Unnecessary complexity:**
- Running YOLO at fixed Hz anyway (1 FPS typical, adaptive by camera state)
- Detector confidence + bbox persistence IS the filter
- High confidence (>0.5) + 3+ frames = real agent
- No need for separate motion scoring complexity

**Direct detection approach:**
- Run YOLOv8 at adaptive rate per camera state (see § 4 Inference Scheduler)
- Standby: ~0.15 FPS (1 in 10 frames)
- Armed: ~3 FPS (3 in 10 frames) — increased for faster event confirmation
- Active: ~10 FPS (all frames)
- Post: ~10 FPS

## 4. Inference Scheduler (Budget + Adaptive Rates)
- Global AI budget: `max_inferences_per_second` (MIPS).
- Per-camera inference ratio `r_c ∈ [0,1]` by state:
  - Standby ≈ 0.15 (1 in 10 frames @ 10 FPS)
  - Armed ≈ 3.0 (3 in 10 frames) — front of queue for faster confirmation
  - Active ≈ 10.0 (all frames)
  - Post ≈ 10.0 (all frames, tail capture)
  - Suppressed: scale non-Active ratios by α (0<α<1) while keeping Active ≈ 1.0.
- Scheduler selects frames up to `r_c * 10 FPS` per camera.
- Micro-batching across cameras with latency budget (e.g., ≤ 50 ms per batch), batch size dynamic (e.g., 8–32 frames).
- Backpressure rule: if batch queue latency > L_max or GPU queue depth > D_max, uniformly reduce all non-Active `r_c` by factor α until stable.

## 5. Detector & Tracking
- Model: YOLOv8 "small" variant (e.g., v8n/v8s) for classes {person, vehicle, animal}.
- Class confidence thresholds `θ_det` (initial guess 0.5–0.6, TBD per class).
- Tracking: IoU association between consecutive frames; optionally simple velocity gating.
- Track record: for each class c maintain rolling list of (timestamp, bbox, conf) entries.

## 6. Temporal Confirmation (3-Second Sustained Rule)
- Define a track τ as successive associated detections of the same class c.
- Allow small gaps: cumulative gap tolerance `g` (0.2–0.4 s TBD).
- Let contiguous segments sum to duration `Σ segments` ignoring gaps ≤ g.
- Promotion criterion: if `Σ segments ≥ 3.0 s`, camera transitions to Active for that track's class.
- Single high-confidence spike does *not* promote unless ≥ θ_confirm (TBD) AND motion_score supports sustained movement logic.

## 7. Camera States & Transitions
States: **Standby → Armed → Active → Post** (plus **Suppressed** system override).

| Transition | Condition (TBD thresholds) |
|------------|---------------------------|
| Standby → Armed | One detection `> θ_arm` |
| Armed → Active | `k` detections within `T_confirm` OR one detection `> θ_confirm` OR sustained track ≥ 3.0 s |
| Armed → Standby | No detections for `arm_expire_s` |
| Active → Post | No detections `> θ_active` for `quiet_window_s` |
| Post → Active | Detection `> θ_active` during Post |
| Post → Standby | Post elapsed `post_roll_s` without detections |
| Any → Suppressed | Global AI pressure or BI error storm |
| Suppressed → Prior | Pressure cleared and error rate below threshold |

Suggested placeholder ranges (all TBD):
- `k = 2–3`; `T_confirm = 1–2 s`.
- `quiet_window_s = 2–3 s`; `post_roll_s = 10–15 s`.
- Confidence chain: `θ_arm < θ_active < θ_confirm`.

## 8. Multi-Camera Concurrency & Grouping
- Multiple cameras may be Active concurrently; events on different property sides coexist.
- Event Group: formed when a camera becomes Active; includes first-degree neighbors if their Armed/Active time is within `group_merge_s` or plausible travel time from the Self-Tuning Travel Matrix.
- Group closure: all member cameras reach Post and complete `post_roll_s` without new Active transitions.

## 9. Archiver & Manifest
- Pre-roll ring per camera: last `R` seconds of sub frames (e.g., 7 s) always in memory.
- Fidelity commitment: persist **all sub frames (10 FPS)** from Armed through Post; persist main frames for Active cameras (neighbor main policy TBD).
- Manifest structure (draft):
```
{
  "camera": "FrontDoor",
  "state_window": {"start_iso": "...", "end_iso": "..."},
  "fps_target": 10,
  "frames": [
     {"ts_iso": "...", "seq": 1234, "profile": "sub", "file": "..._sub.jpg", "inferred": true},
     {"ts_iso": "...", "seq": 1235, "profile": "sub", "file": "..._sub.jpg", "inferred": false}
  ],
  "gaps": 0,
  "profiles": ["sub", "main"],
  "tbd": {"pruning_policy": null}
}
```
- Directory layout examples:
  - `storage/frames/<camera>/<YYYYMMDD>/<HH>/YYYYMMDD_HHMMSS_mmm_sub.jpg`
  - `storage/clips/<event_or_tracking_id>/clip_manifest.json`

## 10. Reconstruction
- MP4 builder (optional) converts manifest frames to video (no audio), constant FPS; duplicates nearest neighbors for missing frames.
- Viewer can stream frames directly from manifest for minimal latency.

## 11. Monitoring & Safeguards
Per-camera metrics:
- `inference_rate`, `detection_rate`, `state`, `queue_depth`, `dropped_frames`.
System metrics:
- GPU/CPU utilization, batch latency distribution, BI error counts, suppression activations.
Health actions:
- Escalate to Suppressed if `inference_queue_latency > L_crit` or `error_rate > E_crit`.
- Auto-tune detection confidence thresholds nightly based on false positive rates.
- Record drift vs BI-provided timestamps if accessible.

Telemetry persistence policy (metadata vs frames)
- During Armed→Active→Post, persist: track summaries for all objects, continuation (CCC) candidates, and co-detections.
- Per-frame detection logs are optional and sampled: up to 2 FPS per Active camera (config), 0.5 FPS in Armed; full 10 FPS not required because tracks capture kinematics. Sampling can be raised temporarily during debugging.
- Decision panels and labels are retained indefinitely for audit/undo; other raw metadata follow retention in `DATA_RETENTION_AND_STORAGE.md`.

## 12. Explicit TBDs
- Pruning/retention policy.
- BI main/sub selection convention.
- Neighbor main-frame saving policy.
- Exact numeric thresholds (confidence, durations, gaps, ratios) per class & camera.
- Travel-time matrix integration details (update cadence, initial defaults).

## 13. Initial Validation Plan (Outline)
1. Simulated frame feed (synthetic motion + injected detections) to verify transitions.
2. Stress test GPU budget with many Standby cameras and a few Active.
3. Measure ring buffer flush latency and manifest write time.
4. Confirm reconstruction correctness (frame count, ordering, gap fill).

## 14. Perspective-Aware Grid (Configurable; Viewer defaults to 12×12)
Purpose
- Improve localization and adjacency plausibility by accounting for perspective.
- Make the lowest center boxes larger and distant/top/edge boxes smaller without requiring real-world calibration.

Geometry
- Density is configurable (e.g., 12×12 in the live viewer, 14×9 or 14×10 for internal learning).
- Row spacing increases toward the bottom (near field) to reflect perspective; column widths may emphasize the center.
- Normalize weights to partition the full image; compute cell rectangles by cumulative sums.

Parameterization (implementation-backed)
- Vertical perspective: exponential spacing with parameter `k_row > 0` (higher → taller bottom rows, shorter top rows).
- Horizontal convergence (keystone): `horiz_vanish_strength ∈ [0,1]` narrows columns at the top toward the image center.
- Bottom expansion: `bottom_expand_strength ≥ 0` slightly widens columns at the bottom (can extend off-frame to give more top resolution).
- Center emphasis: `col_center_emphasis ∈ [0,1]` widens middle columns relative to edges.
- Density: `rows`, `cols` (viewer defaults to 12×12; internal can use 14×9 equivalent).
- boundary_overlap_pct: 0.25

Cell IDs and mapping
- cell_id = r*14 + c (row-major), also store (r,c) for clarity.
- Map detections by centroid; if bbox overlaps adjacent cell beyond `boundary_overlap_pct`, record auxiliary cell for overlap evidence.

Adjacency & travel-time seeding
- Seed edges only between border-sharing cells across cameras (4-neighborhood preferred; diagonals provisional with penalty).
- Use geometric center distance scaled by row depth as a coarse path cost for plausibility gating; true travel times remain empirical (learned).

Config sketch
```
grid:
  rows: 12            # viewer default (internal can be 14x9)
  cols: 12
  k_row: 1.6          # vertical perspective (bigger at bottom)
  horiz_vanish_strength: 0.30
  bottom_expand_strength: 0.15
  col_center_emphasis: 0.20
  boundary_overlap_pct: 0.25
```

Viewer alignment
- The sandbox camera configurator overlays this grid live using the same parameters, with per-camera overrides for “long-distance” views (e.g., BI shortnames d5442ip48/49/50).
- A runtime toggle (`g`) switches the grid on/off for comparison and CPU tuning.

## 15. Face Pipeline (Minimal Input Draft)
Purpose
- Extract high-quality face crops automatically from person detections with minimal user involvement.
- Produce embeddings & panels to aid identity clustering while everyone starts as unknown.

Trigger Conditions
- Only consider person detections with: `confidence >= θ_face_candidate` AND bbox height >= `min_person_px`.
- Prefer main profile frames (higher resolution). If main is unavailable, use sub only if expected face region height ≥ `min_face_px_sub`.

Crop Strategy (Upper Third Heuristic)
- Assume face resides roughly in the upper third of the person bbox; define face ROI:
  - `x1 = person_x1 + pad_w`, `x2 = person_x2 - pad_w`
  - `y1 = person_y1 + pad_top`, `y2 = person_y1 + (person_h * upper_frac)` where `upper_frac ≈ 0.40`.
- If a dedicated face detector returns a face box inside person bbox, prefer that; fallback to heuristic upper-third crop when detector is absent or low confidence.

Temporal & Spatial Diversity
- Sample at most `face_samples_per_sec_active` (e.g., 2) for Active cameras.
- Enforce minimal temporal spacing `Δt_min` between accepted face crops for the same track (e.g., ≥ 0.4 s).
- Reject new crop if IoU with last accepted crop’s face ROI ≥ `iou_redundant_thresh` (e.g., 0.85) to encourage diversity.

Quality Scoring (Per Crop)
- Components: blur (variance of Laplacian), brightness, contrast, pose (yaw/pitch/roll), occlusion (fraction of face covered), face detector confidence.
- Quality formula (example):
  - `Q = w_det*det_conf + w_blur*norm_blur + w_pose*(1 - pose_penalty) + w_light*light_score - w_occl*occlusion_ratio`
- Accept if `Q >= Q_min`.
- Keep top-K (e.g., K=5) per track; others discarded.

Embedding & Track Representation
- Compute embedding for each accepted crop using a face model (ArcFace-like) when `embedding_enabled`.
- Track embedding: robust mean of embeddings after removing outliers (distance > `embedding_outlier_sigma` from provisional mean).

Early Identity Handling (All Unknown Initially)
- Label every face crop as `unknown` until clustering stage has enough support (outside scope of this section).
- Store: `{track_id, ts, camera_id, quality, embedding_vector?, crop_path}`.

Panel Generation vs Single Streaming
- Panel approach: build a composite image of up to K best crops for a track plus 1–2 historical exemplars (once identities exist). Benefits: quick human/LLM evaluation of consistency.
- Single-by-single: lower latency, but loses comparative context. Default: generate a panel at track end or upon transition to Post state.
- Config flag `panel_on_post: true`.

Fallback Behavior
- If face detector fails repeatedly but person track remains strong, keep one heuristic upper-third crop (tagged `heuristic=true`).
- If quality never reaches `Q_min`, no panel produced; track remains person-only.

Storage Layout
- `storage/faces/<camera>/<YYYYMMDD>/<track_id>/<ts>_face.jpg`
- Panel: `storage/faces/<camera>/<YYYYMMDD>/<track_id>/panel.jpg`

Config Sketch
```
faces:
  enabled: true
  states: [Armed, Active]
  θ_face_candidate: 0.55
  min_person_px: 120
  min_face_px_sub: 48
  upper_frac: 0.40
  pad_w: 4
  pad_top: 4
  face_samples_per_sec_active: 2
  Δt_min: 0.4
  iou_redundant_thresh: 0.85
  Q_min: 0.55
  w_det: 0.30
  w_blur: 0.20
  w_pose: 0.15
  w_light: 0.15
  w_occl: 0.20
  embedding_enabled: true
  embedding_outlier_sigma: 2.5
  K_per_track: 5
  panel_on_post: true
```

Open Face Pipeline TBDs
- Precise weight tuning for quality formula.
- Face detector model selection (speed vs accuracy trade-off) and fallback ordering.
- Adaptive min_person_px per camera (distance perspective).
- Policy for discarding crops under IR illumination if quality persistently low.
- Privacy controls (blur faces for timeline unless recognized?).

---
This draft guides implementation while deferring policy-dependent numbers. Update TBD sections after first empirical profiling on live data.

## 16. Long-Gap Adjudication (LLM-Assisted, Cheap Path)

Purpose
- Resolve ambiguous continuations after longer occlusions or cross-camera gaps without relying on brittle visual re-ID.
- Invoke an LLM rarely but decisively, with a forced-choice schema.

Definitions
- Same-camera long gap: no association for `Δt ≥ max(10 s, 3× lost_tolerance)` and then a plausible reappearance in exit-side cells or AOIs (Door/Garage).
- Cross-camera long gap: candidate continuation with `Δt > typical_time(edge) + margin` OR `Δt > P90(edge)` but `Δt ≤ discovery_window_s` (e.g., 40 s).

Precedence (face/plate first, LLM fallback)
- If face embeddings exist for both sides and quality is sufficient, compute cosine distance `d`:
  - Accept YES (no LLM) if `d ≤ θ_face_yes` (e.g., 0.60, model-dependent).
  - Accept NO (no LLM) if `d ≥ θ_face_no` (e.g., 0.80) or if strong contradiction exists (sex/size/clothing/plate mismatch).
  - Otherwise proceed to LLM.
- For vehicles, a plate match (normalized string with confidence) accepts YES; plate contradiction accepts NO without LLM.

When to ask (always for long gaps; otherwise only if needed)
- If gating passes (plausible timing on learned edge or AOI crossing) and the candidate set size ≤ 3, and face/plate are inconclusive, construct panels and ask the LLM.

Panel assembly (deterministic)
- For each side (A: source, B: destination): up to K best crops (face preferred; else consistent torso crops) plus one whole-body keyframe.
- Tight crops; exclude background; similar crop sizes/order across A/B.

Prompt and response schema
- System: "You verify if two image panels depict the same person. Decide only on visible subject features; ignore background/perspective. If evidence is insufficient or contradictory, answer CANNOT."
- User content includes: `Δt`, cameras, AOIs (Door/Garage), height bin, clothing color tags (upper/lower), and the two panels.
- Response must be JSON: `{ "answer": "YES|NO|CANNOT", "confidence": 0.0–1.0, "reasons": "<short>" }`.

Acceptance rules
- Accept YES if `confidence ≥ 0.80` and no hard contradictions (e.g., plate mismatch, incompatible clothing).
- Accept NO if `confidence ≥ 0.70` OR a hard contradiction is detected.
- Otherwise record CANNOT; keep the continuation provisional and avoid updating aggregates.

Threshold notes (tunable)
- `θ_face_yes` and `θ_face_no` depend on the embedding model; start with 0.60 / 0.80 and refine after calibration on your cameras.

## 17. AOI-Driven Continuity (Door, Garage, Exit)

Principle
- AOI crossings are strong, structured signals. When a single person is last seen in an AOI and next seen leaving/entering through a logically connected AOI with plausible timing, assume continuity without LLM.

AOIs
- Door: doorway region; optional outside-side hint for in/out.
- Garage: interior doorway/bay region.
- Exit: egress boundary outside the structure/lot (e.g., sidewalk/side gate/curb region).

Rules
- Same-camera continuity:
  - If a person track ends inside Garage AOI and the next person detection appears at the Garage boundary or Exit AOI within Δt ≤ τ_garage_exit (e.g., 10–20 s) and path is monotone across adjacent cells, auto-merge as the same actor.
  - If a person crosses Door AOI and reappears on the opposite side within Δt ≤ τ_door (e.g., 5–10 s), auto-merge with in/out labeling.
- Cross-camera AOI-linked continuity:
  - If a person exits via Door/Garage on Cam A and appears in border-adjacent cells on Cam B aligned with that AOI’s outside edge within the learned Δt window, accept continuation without LLM unless a contradiction exists.

Contradictions (veto)
- Clear face mismatch, plate contradiction (for vehicle-person linkage), or impossible timing invalidate auto-merge and fall back to standard gating (then LLM if needed).

Tuning (defaults for future reference)
- τ_garage_exit: 10–20 s; τ_door: 5–10 s; refined per camera after data.
- AOIs would be defined on the 32×32 grid via configurator and mapped directly to internal cells.

Budgeting & dedup
- Cache asked pairs (hash of panel inputs) to avoid re-asking.
- Global cap exists but may be high; long gaps are eligible by default given low cost.

Storage & audit
- Persist panels and LLM outputs indefinitely alongside the decision so merges remain undoable and never re-queried.
