# MarengoWatcher — Master Rollout Guide

**Purpose:** 100% redundant compilation of architecture, implementation, and rollout sequencing. Single reference for understanding what to build and when.

**Reading this:** Start-to-finish system overview with implementation order. Consolidates BIGPICTURE_V2, AGENT_TRACKING_AND_MERGE, PORTAL_CONFIGURATION, and phase details from other docs.

---

## Architecture Summary

### Core Concept: Agent-Centric Persistent Tracking

**Agents are people/vehicles**, not detections. Tracks are observable windows where we can SEE an agent. Agents persist conceptually even when not visible (under blanket, inside vehicle, in structure).

**Observable evidence only:** No statistical priors ("Person A always drives"). No routine assumptions. Only what's visible in footage or deducible from logical constraints.

**Dual-grid system:**
- **32×32 intra-camera grid:** Track agents within one camera, prevent ID swaps, learn depth per cell
- **8×8 inter-camera grid:** Learn travel times between cameras (4:1 downsampling from 32×32)

**Three confidence uses (only):**
1. **Agent existence:** Is this detection noise or real? (detector confidence + persistence)
2. **Merge decisions:** AUTO_MERGE vs REVIEW_LLM vs REVIEW_HUMAN (evidence counting)
3. **Location uncertainty (Phase 4-5 deferred):** Where is agent when not visible? (LLM with video)

**No weighted formulas. No EMA. No P90 statistics.** Count evidence, use simple thresholds.

---

## System Components

### 1. Frame Acquisition
- **Blue Iris JPEG API** at 2 FPS initially (Phase 0-1), increasing to 10 FPS in Phase 2
- Sub profile always, main during Active state only
- HTTP keep-alive, jittered scheduling, exponential backoff per camera on errors
- Pre-roll ring buffer: last 7-10s always in memory
- Frame storage uses millisecond-precision timestamps (supports arbitrary FPS)

### 2. Detection (No Motion Gate)
**~~Motion gate removed~~** — originally proposed as pre-filter before YOLO to save GPU budget. **Unnecessary complexity:**
- Running YOLO at fixed Hz anyway (adaptive by state and phase)
- Detector confidence + bbox persistence IS the filter
- High confidence (>0.5) + 3+ frames = real agent

**YOLOv8 small** (v8n/v8s) on sub frames at adaptive rate per camera (scales with FPS):
- **Phase 0-1:** 2 FPS acquisition, inference rates scale proportionally
  - Standby: ~0.3 FPS (1 in 7 frames)
  - Armed: ~1.2 FPS
  - Active: ~2.0 FPS (all frames)
  - Post: ~0.8 FPS
  
- **Phase 2+:** 10 FPS acquisition, full inference rates
  - Standby: ~1.5 FPS (1 in 7 frames)
  - Armed: ~6.0 FPS
  - Active: ~10.0 FPS (all frames)
  - Post: ~4.0 FPS

Classes: {person, vehicle, animal}. Confidence threshold: 0.5-0.6 (TBD per class).

### 3. Intra-Camera Tracking (32×32 Grid)
**IoU association** between consecutive frames. Simple velocity gating if needed.

**Anti-ID-swap measures:**
- Spatial continuity: can't teleport across grid (max 2-3 cells per frame)
- Motion vector consistency: direction/speed shouldn't flip wildly
- **NOT height-based** (too jittery frame-to-frame)
- Mark track as `possibly_switched` if swap suspected

**Fixed 32×32 uniform grid:**
- Simple uniform grid (no perspective warping)
- Each cell = frame_width/32 × frame_height/32 pixels
- Stable across camera changes (doesn't invalidate learned data)
- Height calibration per cell handles depth differences

**Height calibration per cell:**
- Learn expected person bbox height per 32×32 cell from observations
- Online convergence: weighted average of observed heights
- Use for depth normalization, NOT for ID swap detection

### 4. Camera States (Agent-Driven)
States: **Standby → Armed → Active → Post**

| State | Trigger | Inference Rate |
|-------|---------|----------------|
| Standby | No activity | ~0.15 FPS |
| Armed | detection > threshold | ~3 FPS (faster confirmation) |
| Active | track sustained ≥ 3s | 10 FPS |
| Post | no detections for quiet_window_s | 10 FPS |

**Key insight:** If person hides (under blanket, inside vehicle), camera stays Active searching, then degrades to Armed/Standby. Agent marked with possible_locations (Phase 4-5).

### 5. Cross-Camera Merging

**Portal configuration (manual YAML):**
```yaml
portals:
  - camera_a: FrontDoor
    camera_b: Driveway
    cell_a: [0, 15]      # 32×32 cell
    cell_b: [0, 16]
    typical_time: 2      # seconds (optional, can learn)
    max_time: 10
    
  - camera_a: Garage
    camera_b: Outside
    cell_a: [5, 10]
    cell_b: [5, 11]
    exit: true           # Exit portal (track termination at property boundary)
```

**Unified Concurrent Matching Algorithm:**

Every 30 seconds, compare all active + recently-ended tracks (within 60s window) across all cameras using **time/space + Re-ID**. Single algorithm handles:
- Overlapping cameras (concurrent tracks)
- Adjacent cameras (sequential tracks through portals)
- Occlusions (person disappears mid-transition)
- Gaps (person pauses between cameras)

**No separate modes** — concurrent matching handles everything.

**Matching logic (hierarchical):**

**Step 1: Time/space filter (primary)**
- Temporal window: Track2.start_time within 60s of Track1.end_time (or concurrent)
- Spatial plausibility: Fine grid distance + time gap fits walking speed (~1.0 m/s)
- Portal checking: Is cell-to-cell transition a known portal? (optimization only)
- Reject if spatially/temporally impossible

**Step 2: Re-ID disambiguation (when multiple candidates survive)**
- Compute Re-ID similarity for remaining candidates
- If best_similarity > 0.85 AND (best - second_best) > 0.15 → match
- If similarities too close → flag `possible_switched_agent` and proceed to human/LLM

**Step 3: Multi-person handling**
- If Track1.max_concurrent_agents > 1 OR Track2.max_concurrent_agents > 1:
  - Re-ID makes best guess at matching
  - Flag `possible_switched_agent = true` (identities may have swapped)
  - Accept ambiguity, exclude from portal timing learning
  - Human/LLM review can correct later

**Key principle:** Time/space narrows candidates. Re-ID breaks ties. If both ambiguous, accept with uncertainty flag.

**Multi-person scenarios (2-in-2-out strategy):**
- Two agents on CameraA → two agents on CameraB
- Re-ID makes best 1:1 matches (even if confidence mediocre)
- Flag both merges as `possible_switched_agent`
- Prevents agent multiplication (better than creating 4 separate agents)
- Human/LLM can split and re-merge if incorrect

**Score candidates (evidence counting, not arithmetic):**
- **Strong evidence:** face match (>0.75), Re-ID match (>0.85), only candidate, portal crossing
- **Weak evidence:** spatial proximity, timing fit, clothing color match

**Thresholds:**
- `strong_evidence >= 2` → AUTO_MERGE
- `strong_evidence >= 1 + weak_evidence >= 1` → REVIEW_LLM
- Otherwise → REVIEW_HUMAN or mark uncertain

**NO formulas like:** `score = w_t·time + w_s·spatial + w_v·visual`  
**NO arithmetic like:** `evidence_score += 0.30 + 0.40 + 0.50`

### 6. Face Recognition (CodeProject.AI)
**Phase 2 automation path** — trainable face library, auto-merge known people without LLM.

**Face crop extraction:**
- Trigger: person detection, confidence ≥ 0.55, main profile preferred
- ROI: upper third of bbox (face heuristic)
- Quality scoring: blur, brightness, pose, occlusion
- Keep top K=5 per track

**Matching:**
- Cosine distance on embeddings
- `d ≤ 0.60` → AUTO_MERGE (no LLM)
- `d ≥ 0.80` → NO_MERGE (contradiction)
- Between → ask LLM

**Library management:**
- Central **identity registry** for all named people and vehicles (single source of truth)
- When user accepts merge, **identity picker** appears with:
  - Dropdown showing recent/common names (auto-sorted by last seen)
  - Search box for existing identities (fuzzy match warnings prevent duplicates)
  - "New person/vehicle..." button for first-time appearances
- Name normalization (case-insensitive, whitespace-trimmed) prevents "John Doe" vs "john doe"
- Face embeddings from validated merge added to identity's library
- After 3-4 validations, system auto-merges that identity without LLM
- Admin merge tool for cleanup if duplicates slip through

**Portal timing learning (clean observations only):**

**Solo track filter:**
- Track metadata includes `solo_track` boolean and `max_concurrent_agents` count
- Solo track = only one agent detected in all frames (no multi-person groups)
- Computed during tracking, stored in database

**Learning filter:**
```python
# Only learn portal timing from clean observations
learn_portal_timing = (
    merge.track1.solo_track and 
    merge.track2.solo_track and 
    not merge.possible_switched_agent and
    merge.validated  # human or high-confidence LLM
)
```

**Why this matters:**
- Group of 3 walks slowly through portal (8 seconds) → don't contaminate stats
- Solo person walks through (2 seconds) → this is true individual timing
- Multi-person merges still accepted for identity linkage (correct agent matching)
- But excluded from portal timing learning (prevents group behavior from widening windows)

**Computational efficiency:**
- Merge decision is O(1): compare track endpoints only (start/end times, fine grid positions)
- NOT O(N²) trajectory matching: no sample-to-sample distance matrices
- Face/Re-ID similarity: single 512-dim dot product (<1ms)
- Concurrent matching: ~100ms every 30 seconds for all active tracks
- Adjacency matrix updates: at validation rate (~1/hour), not frame rate (10 FPS)

### 7. LLM Adjudication (Gemini Flash 2.0)
**Cheap, deterministic fallback** when face/time/space insufficient.

**Forced-choice schema:**
```json
{
  "answer": "YES|NO|CANNOT",
  "confidence": 0.0-1.0,
  "reasons": "<short explanation>"
}
```

**Panel assembly:**
- Up to K best crops per track + keyframe
- Both sides (source + destination)
- Tight crops, minimal background

**Acceptance:**
- YES if confidence ≥ 0.80 and no contradictions
- NO if confidence ≥ 0.70 OR hard contradiction
- CANNOT → keep provisional, no aggregate updates

**Human-first progressive rollout:**

**Week 1-2: 100% human review, no LLM**
- All merge candidates queued for human decision
- Build ground truth dataset (100+ validations)
- User learns system patterns, calibrates expectations

**Week 3-4: LLM advisory mode**
- LLM processes all candidates, suggestions visible in UI
- Human makes final decision (can override LLM)
- Track LLM accuracy: % agreement with human
- Display LLM confidence distribution

**Week 6+: Selective auto-accept (conservative)**
- Enable auto-accept for LLM confidence > 0.95 only
- All other candidates still queued for human review
- Monitor false positive rate
- Gradually lower threshold if accuracy sustained

**Month 2+: Face recognition primary**
- Known identities auto-merge via face match (no LLM)
- LLM only for unknown people or face match failures
- Human review rare (only ambiguous cases)

**Key principle:** Gradual trust building based on demonstrated accuracy, not blind automation.

### 8. Vehicle Containers (Phase 3)
**Person-vehicle association** when person enters vehicle.

**Evidence (count, don't score):**
- **Strong:** recency (last to enter), face visible in windshield, only candidate
- **Weak:** spatial proximity, timing fit

**Thresholds:**
- `strong >= 2` → AUTO_MERGE
- `strong >= 1 + weak >= 1` → REVIEW_LLM
- Otherwise → mark uncertain

**Uncertainty resolution:**
- Person exits on foot → rules out vehicle association
- Cascades: ruling out Person B increases confidence for Person A

**Acceptable errors:** "In garage" vs "in car" (both "inside structure") — low impact, self-corrects when person emerges.

### 9. Re-ID (Phase 4-5 Deferred)
**Visual embeddings** per camera for fallback when face match fails.

**Use cases:**
- Vehicles (no face)
- People with face uncertainty
- Cross-session tracking (same clothing)

**Build opportunistically:**
- Learn from validated merges (human/LLM approved)
- Per-camera fingerprints (perspective consistent)
- 512-dim embeddings

**Deferred because:**
- Face recognition (CodeProject.AI) already available
- Trainable face library faster path to automation
- Re-ID useful but not required for Phase 2-3

---

## Phase-by-Phase Rollout

### Phase 0: Foundation
**Goal:** Prove acquisition and detection work before tracking complexity.

**Figure-out:**
- Blue Iris JPEG fetcher (2 FPS initially, will increase to 10 FPS in Phase 2)
- YOLOv8 detection pipeline (no motion gate)
- Frame storage (ring buffer + persistence, millisecond-precision timestamps)
- Manifest generation (atomic writes)
- Configuration system (stages, hot-reload)

**Hard (tune by observation):**
- Confidence thresholds per class
- Day/night robustness (IR glare, rain)
- GPU budget allocation (batch size, latency)

**Success criteria:**
- 2 FPS sustained from all cameras for 24 hours
- No dropped frames or BI errors
- Detection logs show real agents, not noise

**Tools:**
- Configuration viewer (sandbox_cameraconfig/camera_dashboard_4x5.py)
- Grid overlay tool (grid_overlay.py)
- Frame viewer with YOLO output

**Note:** Starting at 2 FPS (DEV_SLOW stage) to prove system on limited resources. Will increase to 10 FPS in Phase 2 when face recognition and portal crossing detection benefit from higher temporal resolution.

---

### Phase 1: Single-Camera Tracking
**Goal:** Prove intra-camera tracking before cross-camera merging.

**Figure-out:**
- IoU tracking with velocity gating
- 32×32 perspective-aware grid
- Height calibration per cell (online learning)
- Agent creation (detector confidence + persistence)
- ID-swap detection (spatial continuity, motion vectors)
- Camera state machine (Standby → Armed → Active → Post)

**Hard (tune by observation):**
- IoU threshold for association
- Max cell jumps per frame (2-3 typical)
- Height calibration convergence rate
- ID-swap sensitivity (false positives vs misses)
- State transition timing (arm_expire_s, quiet_window_s, post_roll_s)

**Success criteria:**
- Track person across one camera for 60+ seconds through occlusion
- No ID swaps during clean passes
- False ID-swap flags < 5%
- Camera states transition correctly

**Tools:**
- Single-camera viewer (sandbox_cameraconfig/test_single_camera.py)
- Track visualization (bbox + grid cell overlay)
- State transition logger

---

### Phase 2: Cross-Camera Correlation
**Goal:** Prove merge logic and portal configuration.

**Figure-out:**
- **Increase FPS to 10** (from 2 FPS in Phase 0-1) — face recognition and portal crossing benefit from higher temporal resolution
- 8×8 downsampling from 32×32 (4:1 clean mapping)
- Portal configuration (YAML format)
- Portal configuration editor UI (visual tool for marking doors/gates)
- **Identity registry** (central table for all named people/vehicles)
- **Identity picker UI** (dropdown + search + fuzzy match warnings)
- find_merge_candidates_via_portals()
- Evidence counting (strong/weak)
- Face crop extraction (upper third heuristic)
- CodeProject.AI wrapper (face recognition)
- Face library linked to identity registry (prevent duplicate names)
- Panel builder (2×6 layout, 768×768)
- LLM integration (Gemini Flash 2.0)
- LLM forced-choice schema
- Manual review UI with identity picker (queue + naming + corrections)

**Hard (tune by observation):**
- Portal time windows (typical_time, max_time)
- Face recognition thresholds (cosine distance)
- Face quality gating (blur, pose, occlusion)
- LLM confidence acceptance (0.80 for YES, 0.70 for NO)
- Evidence counting thresholds (strong >= 2 → auto-merge)

**Success criteria:**
- **10 FPS sustained from all cameras** (up from 2 FPS)
- Merge person across 2 cameras with portal
- Face recognition handles 60%+ of known-person merges
- 50%+ auto-merge rate after 1 week of operation
- LLM adjudication works with forced-choice schema
- Manual review UI allows corrections, system learns

**Tools:**
- Multi-camera viewer (2×2 grid, synchronized playback)
- Merge decision logger (evidence breakdown per candidate)
- Face library manager (label unknown faces)
- LLM panel viewer (see what LLM saw)
- Review queue UI
- Portal configuration editor (click cells to define connections)

---

### Phase 3: Uncertainty Handling
**Goal:** Prove multi-state tracking and vehicle containers.

**Figure-out:**
- possible_locations tracking (in_vehicle, in_structure, unknown)
- Vehicle association (person enters vehicle)
- Evidence counting for vehicle containers
- Exit portals (track termination at property boundary)
- Uncertainty resolution (process of elimination)
- Manual review UI for vehicle associations

**Hard (tune by observation):**
- Vehicle association thresholds
- Uncertainty collapse logic (ruling out candidates)
- Exit portal time windows
- Location confidence decay (how long before "unknown"?)

**Success criteria:**
- Handle "2 people + 1 car" gracefully
- Track person entering vehicle, associate correctly
- Exit portals terminate tracks at property boundary
- Resolve uncertainty via later observations (person exits on foot)
- Face recognition library covers regular visitors

**Tools:**
- Vehicle container viewer (show person → vehicle associations)
- Uncertainty dashboard (possible_locations visualization)
- Exit portal logger

---

### Phase 4: Maturity
**Goal:** Tighten portal windows, improve confidence, add Re-ID fallback.

**Figure-out:**
- Portal window shrinking (learn tighter typical_time from validated merges)
- Correction feedback loops (learn from manual corrections)
- Panel pruning (discard low-quality panels)
- (Optional) Re-ID database building from validated merges
- (Optional) Re-ID matching pipeline
- Confidence decay (stale portals without recent observations)

**Hard (tune by observation):**
- Portal learning rate (how fast to tighten windows)
- Re-ID threshold calibration (cosine distance for vehicles/people)
- Re-ID model selection (speed vs accuracy)
- Confidence decay rate (exponential? linear?)

**Success criteria:**
- 90%+ auto-merge rate
- Tight portal windows (typical_time ± 1-2s)
- Low false merge rate (<2%)
- Face recognition + time/space handle most cases
- Re-ID as fallback for vehicles/face-match failures

**Tools:**
- Portal statistics dashboard (time distributions, confidence trends)
- Re-ID visualization (embedding space, nearest neighbors)
- Confidence calibration plots

---

### Phase 5: Nice-to-Haves
**Goal:** Polish and advanced features (only if needed).

**Figure-out:**
- Identity clustering (group "unknown" faces)
- LPR (CodeProject.AI license plate recognition)
- Re-ID matching optimization
- AOI/portal semantics (room-level tracking)
- Long-gap adjudication improvements

**Hard (tune by observation):**
- Clustering thresholds (how tight for same person?)
- LPR confidence thresholds
- Portal transit time refinement
- Re-ID diversity per person (multiple embeddings)

**Success criteria:**
- Regular visitors auto-identified without manual labeling
- LPR tracks vehicles across property
- Complex scenarios handled (3+ people, 2+ vehicles)
- System runs autonomously with minimal human review

**Tools:**
- Identity clustering UI (merge unknown faces)
- LPR dashboard (vehicle tracks + plates)
- AOI configuration UI (room-level tracking)

---

## Implementation Order (Detailed)

### Phase 0 Tasks (Foundation)
1. Blue Iris JPEG fetcher (HTTP API wrapper)
2. Configuration system (YAML, stages, hot-reload)
3. Frame storage (ring buffer in memory, persistence to disk)
4. YOLOv8 detection pipeline (load model, batch inference)
5. Manifest generation (JSON, atomic writes)
6. Resource throttling (adaptive frame rate by camera state)
7. Monitoring (inference rate, queue depth, errors)

**Tools:** Configuration viewer, grid overlay, frame viewer

**Success:** 10 FPS sustained for 24 hours

---

### Phase 1 Tasks (Single-Camera Tracking)
1. 32×32 grid generation (perspective-aware, per-camera params)
2. IoU tracking (associate detections across frames)
3. Height calibration (online learning per cell)
4. Agent creation (detector confidence + persistence ≥ 3s)
5. ID-swap detection (spatial continuity, motion vectors)
6. Camera state machine (Standby/Armed/Active/Post)
7. Track persistence (maintain agent across gaps)

**Tools:** Single-camera viewer, track visualization, state logger

**Success:** Track person for 60+ seconds, no ID swaps

---

### Phase 2 Tasks (Cross-Camera Correlation)
1. 8×8 downsampling (4:1 mapping from 32×32)
2. Portal configuration (YAML format, load/validate)
3. find_merge_candidates_via_portals() (lookup candidates by portal)
4. Evidence counting (strong/weak, thresholds)
5. Face crop extraction (upper third heuristic, quality scoring)
6. CodeProject.AI wrapper (face recognition API)
7. Face library management (store embeddings, label unknowns)
8. Panel builder (2×6 layout, 768×768)
9. LLM integration (Gemini Flash 2.0, forced-choice schema)
10. Manual review UI (queue, corrections, learning)

**Tools:** Multi-camera viewer, merge decision logger, face library manager, review queue UI

**Success:** 50%+ auto-merge, 60%+ face recognition coverage

---

### Phase 3 Tasks (Uncertainty Handling)
1. possible_locations tracking (in_vehicle, in_structure, unknown)
2. Vehicle association (person enters vehicle, evidence counting)
3. Exit portals (track termination at property boundary)
4. Uncertainty resolution (process of elimination, cascades)
5. Vehicle container UI (manual review for associations)

**Tools:** Vehicle container viewer, uncertainty dashboard, exit portal logger

**Success:** Handle "2 people + 1 car" gracefully, resolve via observations

---

### Phase 4 Tasks (Maturity)
1. Portal window shrinking (learn tighter times from validated merges)
2. Correction feedback loops (learn from manual corrections)
3. Panel pruning (discard low-quality panels)
4. (Optional) Re-ID database building
5. (Optional) Re-ID matching pipeline
6. Confidence decay (stale portals)

**Tools:** Portal statistics dashboard, Re-ID visualization, confidence calibration plots

**Success:** 90%+ auto-merge, tight portal windows, low false merge rate

---

### Phase 5 Tasks (Nice-to-Haves)
1. Identity clustering (group unknown faces)
2. LPR (CodeProject.AI wrapper)
3. Re-ID optimization
4. AOI/portal semantics (room-level tracking)
5. Long-gap adjudication improvements

**Tools:** Clustering UI, LPR dashboard, AOI configuration UI

**Success:** Autonomous operation, minimal human review

---

## Key Design Principles

### 1. Codifiable and Observable
**Count evidence, don't do arithmetic.**
- Can't debug why weights are 0.30 vs 0.40
- CAN debug "face match = strong evidence"

### 2. Configuration Over Learning
**For known structure (portals), configure it.**
- User knows where doors are
- Manual YAML beats statistical discovery
- Learning is for timing refinement, not structure

### 3. Three Confidence Uses Only
1. Agent existence (detector confidence + persistence)
2. Merge decisions (evidence counting)
3. Location uncertainty (Phase 4-5 deferred, LLM with video)

**NOT for:**
- Weighted scoring formulas
- Statistical priors
- Routine assumptions

### 4. Face Recognition First
**CodeProject.AI is already available, trainable, mature.**
- Faster path to automation than Re-ID
- Handles known people without LLM
- Re-ID deferred to Phase 4-5 as fallback

### 5. LLM as Cheap Fallback
**Gemini Flash 2.0 is cheap, forced-choice is deterministic.**
- Only when face/time/space insufficient
- Forced-choice schema (YES/NO/CANNOT)
- Learn from corrections

### 6. Acceptable Errors
**"In garage" vs "in car" (both "inside structure") — low impact.**
- Will self-correct when person emerges
- Don't over-engineer uncertainty resolution
- Process of elimination handles most cases

### 7. Portal Timing (Window Tightening Only)
**User configures portals. System learns timing to reject false positives.**

**Why we DO this:**
1. **False positive rejection:** Without timing, Track A ending at Front Door + Track B starting at Driveway 45s later = wasted LLM call. Learning "typical=2s, max=10s" rejects the 45s case instantly.
2. **Confidence ordering:** When 3 tracks all start within 15s, which to LLM-review first? The one closest to learned portal timing.
3. **Cheap insurance:** Rolling statistics (mean, P50, P90) from validated merges. No EMA, no complexity. Once 50 observations collected, never waste LLM budget on implausible candidates.

**What we DON'T do:** Portal discovery (user knows where doors are). Just timing refinement.

**Optional:** If face recognition handles all cases, portal timing becomes irrelevant. But for vehicles or face-occluded people, it's the only filter before LLM.

### 8. No Motion Gate
**Detector confidence + persistence IS the filter.**
- Running YOLO at fixed Hz anyway
- High confidence (>0.5) + 3+ frames = real agent
- Motion gate is unnecessary pre-filter complexity

---

## Why This Ordering

1. **Data capture first** (can't learn without frames)
2. **Prove one thing at a time** (intra before inter)
3. **Simple heuristics before complex** (confidence before Re-ID)
4. **Face recognition before Re-ID** (already available, trainable)
5. **Defer nice-to-haves** until core proven (AOIs/portals Phase 5)
6. **Fast learning on validated merges** (trust human/LLM approvals)

---

## Reference Documents

**Core architecture:**
- `BIGPICTURE_V2.md` — System overview, components, data flow
- `ARCHITECTURE_UPDATES_NOV2025.md` — Agent-centric model, dual-grid system
- `AGENT_TRACKING_AND_MERGE.md` — Merge logic, evidence counting, vehicle containers
- `PORTAL_CONFIGURATION.md` — Camera connections, YAML format

**Specifications:**
- `EVENT_INFERENCE_SPEC.md` — Detection, tracking, face pipeline, LLM adjudication
- `CONFIG_AND_LEARNING_UI.md` — Configuration UI, manual review, corrections
- `EXTERNAL_AI_INTEGRATIONS.md` — CodeProject.AI (face recognition, LPR)

**Implementation:**
- `IMPLEMENTATION_GUIDE.md` — Phase 1 implementation (MQTT logger, Blue Iris API)
- `HARD_VS_FIGURE_OUT.md` — Task breakdown with phase tags
- `DATA_RETENTION_AND_STORAGE.md` — Manifests, pruning, retention policy

**Supporting:**
- `NOMENCLATURE.md` — Terminology (agent, track, portal, merge)
- `UI_ARCHITECTURE.md` — UI tools per phase
- `README.md` — Documentation index

---

## Status

**Current state:** Phase 0-1 planning complete. No code written yet beyond sandbox exploration.

**Next steps:** Begin Phase 0 foundation work — Blue Iris JPEG fetcher, YOLOv8 pipeline, frame storage.

**Validation:** 10 FPS sustained for 24 hours from all cameras without dropped frames.

---

## Notes

- **This document is 100% redundant by design** — consolidates all architectural decisions and implementation details into single reference.
- **Living document** — update as implementation reveals better approaches.
- **Priorities may shift** — what breaks first in real footage may reorder tasks.
- **Observable evidence only** — no statistical priors, no routine assumptions.
- **Configuration over learning** — user knows structure (doors), system learns timing.
- **Count evidence, don't do arithmetic** — codifiable and observable.

---

**Last updated:** 2025-12-19
