# Big Picture V2 — JPEG-only, 10 FPS, inference-scheduled

This document captures the new high-level architecture. It leaves the original BigPicture intact and focuses on the revised plan where Blue Iris (BI) provides only JPEG snapshots and we reconstruct any needed clips from our saved frames. Pruning policy is explicitly TBD and will be set after the logic is finalized.

## Goals
- Build a Golden timeline by ignoring noise and only promoting “real” activity.
- “Real” = a well-identified object (person, vehicle, animal) moving for ≥ 3.0 seconds.
- Maintain playback fidelity: once we commit to an event, saved frames reconstruct smooth 10 FPS playback.

## Operating assumptions
- Acquisition starts at 4 FPS per camera (JPEG snapshots), scaling to 10+ FPS as system matures.
- Profiles:
  - sub (low-res): always available and used for detection.
  - main (high-res): captured during detail windows for Active cameras (neighbors TBD).
- No video/audio requests to BI; BI is connection manager only.

## Grid Systems (Adaptive Learning Grid)

### 32×32 Intra-Camera Grid with Learned Transition Times

**Purpose:** Fine-grained spatial tracking within each camera view with adaptive temporal thresholds.

**Design:**
- Each camera view partitioned into 32×32 uniform grid (1024 cells)
- Each cell stores **learned transition time** (seconds to reach adjacent cell)
- Perspective-aware initialization: 3s/cell (bottom/near) → 12s/cell (top/far)
- Updates from confirmed track continuations (IoU tracker + grid matches)
- Exponential moving average (α=0.3) for aggressive adaptation

**Spatial Indexing:**
- Detection bbox centroid → grid cell (x, y)
- Cell size: ~22×15 pixels (704×480 sub profile)
- Real-world: ~0.5-1 meter per cell (camera-dependent)

**Temporal Thresholds:**
- Baseline: distance × learned_time_per_cell × 3.0 (generous multiplier for stopping/turning)
- Edge cells: 2× bonus for entering/exiting scenarios
- Example: Center adjacent cells ~22s, corner-to-corner ~999s

**Learning Constraints:**
- Only updates from confirmed matches (IoU tracker continuations)
- Rejects teleportation: movement < 0.5s/cell
- No upper limit: allows standing/napping indefinitely
- Safety: confirmed_match=True required to prevent cross-object pollution

**Key Assumption:** Objects cannot teleport >1-2 cells per sampling period (250ms @ 4 FPS)

### 8×8 Inter-Camera Grid (Phase 2)

Coarser 8×8 grid (64 cells) for learning adjacency between cameras:
- Clean 4:1 downsampling from 32×32 (`row_8 = row_32 // 4`)
- Edges between cells track travel times
- Enables neighbor boost when agent moves between cameras
- Tractability: 4K edges per camera pair vs 1M for full resolution

This dual-grid approach provides spatial precision (32×32 tracking) while maintaining computational tractability (8×8 cross-camera learning).

## Agent-Centric Model

**Core concept:** Track persistent agents (people, vehicles) conceptually, not just camera-visible segments.

### Agent Lifecycle
- **Created:** When new detection appears with no plausible continuation
- **Visible:** Direct camera observation (track segments with bbox, cell, motion)
- **Hidden:** Not currently visible (possible locations: in vehicle, in structure, off-camera)
- **Merged:** Multiple track segments determined to be same agent
- **Terminated:** Agent leaves property or becomes inactive for extended period

### Agent State
Each agent maintains:
- **Track segments:** All camera-observable detections with timestamps, cells (32×32), motion vectors
- **Current state:** Visible (camera + cell) or Hidden (possible_locations with confidence scores)
- **Identity:** Unknown-N initially; **promoted via face recognition (CodeProject.AI trainable library)** or LLM confirmation
- **Face library:** Best face crops registered in CodeProject.AI for automatic recognition on future appearances
- **Image bank:** Best quality images per camera × lighting condition for re-identification fallback
- **Merge history:** Associations with other track segments, confidence, evidence used (face match, time/space, LLM)

### Multi-State Location Tracking
Agents can have uncertain locations with confidence scores:
```
Agent A possible_locations:
  - in_vehicle (V1, left_property): 0.70
  - in_structure (garage): 0.25
  - unknown: 0.05
```
Later observations collapse uncertainty (e.g., person exits on foot → rules out vehicle association).

## Roles & responsibilities
- Blue Iris: camera connection manager and JPEG provider (main/sub). No MP4/video.
- Marengo (this system): detection, tracking, agent persistence, merge decisions, archiving, reconstruction, and correlation.

## Architecture overview
1. **Acquisition:** Every camera produces sub JPEG frames at 4 FPS initially (scaling to 10+ FPS). Main frames requested only during detail windows.
2. **Motion Gate (cheap):** Assigns motion scores and filters trivial frames before AI.
3. **Inference Scheduler (budget-driven):** Chooses which frames get detection based on camera state and global AI budget; batches across cameras.
4. **Detector (CodeProject.AI):** Finds people, vehicles, animals on sub frames.
5. **Intra-Camera Tracking:** 
   - IoU tracker links sequential detections (simple overlap matching)
   - 32×32 adaptive grid provides spatial indexing + learned temporal thresholds
   - Grid cells learn "seconds to adjacent cell" from confirmed track continuations
   - Perspective-aware: bottom cells fast (3s), top cells slow (12s), updates aggressively (α=0.3)
6. **Agent Merger (Phase 2):** Match track segments across cameras using 8×8 inter-grid, learned travel times, face/appearance, and LLM adjudication.
7. **State Machine per camera:** Standby → Armed → Active → Post (driven by agent activity in view).
8. **Archiver & Manifest:** Pre-roll buffer; persist all frames for Armed/Active/Post cameras; write manifests for reconstruction.
9. **Playback & Timeline:** Reconstruct clips from JPEGs; display agent continuity across cameras.

## Event definition (high-level)
- Promote camera to Active only if an agent is detected and moving for at least 3.0 seconds (with small gap tolerance).
- Multi-camera concurrency is expected; multiple cameras can be Active at once tracking different agents.
- Events are agent-centric: "Person A entered property, moved through CamA→CamB→CamC, exited via Side Door"

## Camera state model (Agent-Driven)
Camera state is determined by agents in view:

- **Standby:** Quiet baseline. 10 FPS acquired, pre-roll ring only. Sparse inference (~10% sampling).
- **Armed:** Recent activity or neighbor camera is Active. Begin persisting frames. Medium inference (~60% sampling).
- **Active:** At least one agent confirmed (≥3s sustained movement). Persist all frames (10 FPS sub + main). High inference (100% sampling).
- **Post:** Cool-down after agent leaves frame. Continue persistence through post-roll. Medium inference.
- **Suppressed:** System pressure. Maintain fidelity for Active cameras; reduce others.

**Neighbor boost:** When CamA goes Active, adjacent cameras (via learned 8×8 edges) elevate to Armed.

**Agent under blanket scenario:** If agent disappears for 2 minutes:
- Camera remains Active initially (searching)
- Degrades to Armed after quiet threshold
- Eventually Standby if no re-detection
- Agent state: marked with possible_locations (hidden, possibly moved)

## Fidelity commitment
- From Armed through Post, persist all sub frames at 10 FPS for those cameras to guarantee smooth playback. Main is persisted during Active (neighbors TBD).

## Agent Merge Hierarchy

Decisions made in priority order (strongest → weakest evidence):

1. **Time/Space Gating (8×8 inter-grid, learned edges)**
   - Known overlap (Δt ≈ 0, validated) → AUTO-MERGE
   - Within learned travel time window → CANDIDATE

2. **Face Recognition (CodeProject.AI trainable library)** [Phase 2]
   - Face match confidence > 0.70 → AUTO-MERGE (known person)
   - Unknown face → continue to other evidence
   - Poor face quality → skip face check

3. **Observable Evidence**
   - Spatial proximity + timing → High confidence
   - Only candidate (no ambiguity) → MERGE

4. **LLM Adjudication (Gemini Flash 2.0, cheap)** [Phase 2]
   - Multiple candidates, medium confidence → ASK
   - Present image panels + context → YES/NO/UNCERTAIN

5. **Human Review (fallback)**
   - Low confidence or LLM uncertain → QUEUE for manual decision
   - Feed decision back to strengthen time/space model AND register faces in library

**Learning flywheel:** 
- Initially ~50% auto-merge (wide windows, low confidence, empty face library)
- Face library builds from validated merges (human/LLM confirms → register faces)
- After weeks: 90%+ auto-merge via face recognition (known people) + time/space (tight windows)

## Storage & Retention
- **Frames (9 days):** All sub frames (10 FPS) for Armed/Active/Post cameras. Main frames during Active (optional).
- **Metadata (indefinite):** Agent tracks, merge decisions, edges, corrections, image banks, face recognition results.
- **Face library (indefinite):** CodeProject.AI trainable face recognition library; faces registered on validated merge; multiple faces per person across cameras/lighting.
- **Image bank (indefinite):** Top 3 quality images per agent × camera × lighting condition for manual re-ID if needed.
- **Re-ID fingerprints (indefinite, Phase 4-5 optional):** Per-camera visual embeddings (512-dim) for fallback when face match uncertain. Especially effective for vehicles; useful for people within session. Deferred until face recognition proven.
- **Height calibration (indefinite):** Per camera × cell_32x32, updated online from observations.
- **Grid snapshots (12 weeks rolling):** Weekly automated backups of learned edges (8×8 inter-camera) and height calibration for recovery and visualization.

## Configuration & Stages
See `CONFIGURATION_SYSTEM.md` for full details.

**Operational Stages:** Control resource usage and feature complexity based on hardware capability and development phase.
- **DEV_SLOW:** Underpowered hardware (2 FPS, 5 cameras, no merging, no external AI) — proves Phase 0-1 logic without taxing resources.
- **DEV_FULL:** Full development system (10 FPS, all cameras, merging enabled, face recognition/LPR on select cameras) — Phase 0-3 testing.
- **PROD_LEARNING:** Production learning mode (conservative auto-merge 0.85, human review queue active) — Phase 3-4 with human-in-loop.
- **PROD_AUTO:** Mature production (aggressive auto-merge 0.70, minimal human review) — Phase 5 autonomous operation.

**External AI Services:** Multiple CodeProject.AI servers registered with URL, priority, model checkboxes (face_recognition, lpr); scheduler tracks response times, queue depth, CPU state for intelligent routing and failover.

**Hot-Reload:** Configuration changes applied without restart; version stamping on artifacts; automatic rollback if metrics degrade.

## UI Architecture
See `UI_ARCHITECTURE.md` for full details.

**Strategy:** Separate, focused applications per phase (config, viewing, training, timeline). Consolidate later only if needed.

**Phase 0-1 UIs:**
- Camera Grid Viewer (tkinter) — live feeds, 32×32 grid overlay, ignore zones
- CLI Health Monitor (Click) — system health, config management, secrets
- Single-Camera Debug Viewer (tkinter) — track visualization, cell assignment, height calibration

**Phase 2-3 UIs:**
- Merge Decision Queue (web) — review merge candidates, provide ground truth
- Edge Visualization Dashboard (web) — visualize learned edges, confidence growth
- Face Library Manager (web) — manage registered faces, correct identities

**Phase 4+ UIs:**
- Event Timeline Viewer (web) — browse activity, play reconstructed clips
- Configuration Studio (web, optional) — adjacency hints, AOI placement, camera flags

## Open items (explicitly TBD)
- Exact confidence thresholds for merge decisions (will calibrate from real data).
- Day/night motion gate thresholds.
- Quiet period timings (Active → Armed → Standby transitions).
- BI main/sub selection convention (aliases vs width/height parameters).
- Neighbor detail policy (when to save main for adjacent cameras).
- Portal/structure semantics (deferred to Phase 2).

---

This high-level is the source of truth. Detailed specs define grid mechanics, tracking, merge logic, manifest format, and configuration system.