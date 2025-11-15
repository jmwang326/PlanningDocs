# Level 6 - Deployment Plan

## Purpose
Phased rollout sequence and gates. What must work before moving forward.

---

## Philosophy

### Prove One Thing at a Time
- Phase 0: Frames arrive reliably
- Phase 1: LocalTracks work (single camera)
- Phase 2: Cross-camera merging works
- Phase 3: Uncertainty handling works
- Phase 4: Automation works (grid + face library)
- Phase 5: Polish (optional)

### Human-First, Then Automation
Start with 100% human review. Add automation only after validation.

---

## Phase 0: Acquisition

**Goal:** Frames arrive reliably from Blue Iris

**Components:**
- Blue Iris JPEG fetcher
- YOLO11 detection pipeline
- Frame storage (ring buffer + persistence)
- Configuration system (YAML, hot-reload)

**Success Gate:**
- 24+ hours sustained acquisition (2-4 FPS, all cameras)
- No dropped frames
- YOLO detects real objects (people, vehicles), not noise

---

## Phase 1: Single-Camera Tracking

**Goal:** LocalTracks work on one camera

**Components:**
- ChunkProcessor (12s chunks → LocalTracks)
- TemporalLinker (link tracks across chunks)
- Camera state machine (Standby/Armed/Active/Post)

**Success Gate:**
- Track person across multiple 12s chunks
- State transitions work (Standby → Armed → Active → Post)
- Noise filtered (brief detections not promoted to tracks)
- LocalTracks have correct timestamps, bounding boxes, face crops

---

## Phase 2: Cross-Camera Merging

**Goal:** IdentityResolver finds merge candidates, review queue works

**Components:**
- IdentityResolver (find merge candidates)
- Review Queue UI (human review)
- Face recognition (CodeProject.AI)
- Grid learning (6×6 spatial model)

**Success Gate:**
- Merge candidates appear in review queue
- Human can approve/reject merges via UI
- Face recognition works (≥ 0.75 confidence = auto-merge)
- Grid learning populates from validated merges
- Review queue decreases as face library + grid grow

**Note:** Bootstrap phase (L5_Bootstrap.md) runs concurrently - building face library while system learns.

---

## Phase 3: Uncertainty Handling

**Goal:** Multi-assignment and process of elimination work

**Components:**
- Multi-assignment support (track possibly Agent A or B)
- Vehicle association (person enters vehicle)
- Process of elimination (cascading updates)

**Success Gate:**
- Handle "2 people + 1 car" without breaking
- Multi-assignment persists until evidence resolves it
- Process of elimination cascades correctly (resolving A removes A from other candidates)

---

## Phase 4: Automation

**Goal:** Grid learning + face library reduce review queue to near-zero

**Components:**
- Portal timing refinement (tighten windows)
- Face library coverage (regular visitors)
- LLM adjudication (Gemini Flash)

**Success Gate:**
- 90%+ auto-merge rate
- Review queue < 5 items/day
- Low false merge rate (<2%)
- Known people auto-identified without review

---

## Phase 5: Polish (Optional)

**Goal:** Nice-to-haves

**Tasks:**
- License plate recognition (CodeProject.AI LPR)
- Re-ID optimization (custom training)
- Identity clustering (group unknown faces)

**Success Gate:**
- System handles complex scenarios (3+ people, 2+ vehicles)

---

## Operating Modes

**Online Learning Mode (Initial):**
- Queue all uncertain merges for review
- High face threshold (0.85)
- Conservative grid learning (solo tracks only)
- Human validates all face library additions

**Autonomous Mode (After Bootstrap):**
- Auto-merge high-confidence decisions
- Lower face threshold (0.70)
- LLM reviews uncertain cases
- Human reviews only complex cases

**Transition criteria:**
- Auto-merge rate > 80%
- Face library > 5 key people
- Grid learning > 80% common paths

---

## Risk Management

**Phase 2-3 (High Risk):**
- ID swap (wrong merge breaks timeline)
- Face library pollution (wrong face registration)
- Multi-person merge disambiguation

**Mitigation:** Conservative thresholds, human validation before face registration

**Phase 4 (Medium Risk):**
- Portal timing contamination (group behavior)
- LLM cost overrun

**Mitigation:** Solo track filter, cache LLM results

**Phase 5 (Low Risk):**
- Noise promotion (tree shadow → agent)

**Mitigation:** Timeline review, manual cleanup
