# Level 6 - Deployment Plan

## Purpose
Phased rollout, timelines, and success criteria.

## Rollout Philosophy

### Prove One Thing at a Time
- Phase 0: Acquisition works
- Phase 1: Tracking works (single camera)
- Phase 2: Merging works (cross-camera)
- Phase 3: Uncertainty handling works
- Phase 4+: Optimization and automation

### Human-First, Then Automation
- Week 1-2: 100% human review
- Week 3-4: LLM advisory (human validates)
- Month 2+: Auto-merge known people via face library
- Mature: 90%+ auto-merge

## Phase 0: Foundation (Weeks 1-2)

### Goal
Prove acquisition and detection work before tracking complexity.

### Tasks
- Blue Iris JPEG fetcher
- YOLO11 detection pipeline
- Frame storage (ring buffer + persistence)
- Configuration system (YAML, hot-reload)

### Success Criteria
- Sustained acquisition: 2-4 FPS from all cameras for 24+ hours
- No dropped frames or Blue Iris errors
- Detection logs show real objects (people, vehicles), not noise

### Tools
- Configuration viewer
- Frame viewer with YOLO overlays

---

## Phase 1: Single-Camera Tracking (Weeks 3-4)

### Goal
Prove intra-camera tracking before cross-camera merging.

### Tasks
- YOLO11 tracking integration
- Camera state machine (Standby/Armed/Active/Post)
- Track quality filtering (duration, movement)
- Agent creation (persistent across gaps)

### Success Criteria
- Track person across one camera for 60+ seconds
- State transitions work correctly (Standby → Armed → Active → Post)
- Noise filtered (brief detections not promoted to tracks)

### Tools
- Single-camera debug viewer (track visualization)
- State transition logger

---

## Phase 2: Cross-Camera Merging (Weeks 5-8)

### Goal
Prove merge logic and face recognition.

### Tasks
- Portal configuration (manual YAML)
- Merge candidate finding (time/space gating)
- Face recognition integration (CodeProject.AI)
- Face library management
- LLM adjudication (Gemini Flash)
- Manual review UI (queue, validation, corrections)

### Success Criteria
- Merge person across 2+ cameras via portals
- Face recognition handles 60%+ of known-person merges
- 50%+ auto-merge rate after 1 week of operation
- LLM adjudication works with forced-choice schema

### Human-in-Loop & Adjudication Workflow
The system is designed around a human-centric review process. The user is always in control of the final decision for ambiguous merges. Automation is an opt-in feature, not a default.

**Default Workflow (Human-in-Control):**
1.  **Human Review Queue:** All merge candidates that are not resolved by high-confidence evidence (e.g., a definitive face match) are placed in a manual review queue.
2.  **On-Demand LLM Consultation:** From the review UI, the operator has options for how to use the LLM as a consultant:
    *   The UI can display a pre-computed LLM opinion, which the operator can then accept or override.
    *   For any given case, the operator can click a button to "Ask LLM" to get a fresh analysis on demand.
3.  **The Human is the Final Authority:** The operator's decision is always considered ground truth. It is used to update the agent timeline and provide the high-quality data needed to train the system's other models (face library, `6x6` grid timings).

**Optional Automated Workflow (User Opt-In):**
- **Configuration:** The system will provide a configuration setting (e.g., `review_mode: manual | automatic`) to allow the user to opt-out of the manual review loop for certain cases. This is the mechanism for the user to "be out of the loop."
- **Auto-Acceptance Threshold:** When in `automatic` mode, the system can be configured to automatically accept LLM decisions that meet a user-defined confidence threshold (e.g., `confidence > 0.90`).
- **Phased Trust Building:** The recommended deployment starts in `manual` mode to build a ground-truth dataset and gain confidence in the LLM's accuracy. The user can later choose to enable `automatic` mode for high-confidence cases to reduce their workload.

**Primary Automation Path (Face Recognition):**
- The most effective path to automation is building a robust face library for known individuals.
- Once a person is reliably identified via face recognition, their merges can be automatically approved without needing LLM or human review, representing the highest level of automation.

### Tools
- Multi-camera viewer (2×2 grid, synchronized playback)
- Merge decision logger (evidence breakdown)
- Face library manager
- Review queue UI

---

## Phase 3: Uncertainty Handling (Weeks 9-12)

### Goal
Prove multi-state tracking and vehicle containers.

### Tasks
- Multi-assignment support (track possibly Agent A or B)
- Vehicle association (person enters vehicle)
- Process of elimination (cascading updates)
- Uncertainty visualization in UI

### Success Criteria
- Handle "2 people + 1 car" gracefully
- Multi-assignment persists, resolves later when evidence appears
- Process of elimination rules out candidates correctly

### Tools
- Uncertainty dashboard (multi-assignment visualization)
- Vehicle container viewer

---

## Phase 4: Maturity (Months 3-4)

### Goal
Tighten windows, increase automation.

### Tasks
- Portal timing learning from validated merges
- Face library coverage (regular visitors)
- Re-ID fallback for vehicles (optional)
- Confidence calibration

### Success Criteria
- 90%+ auto-merge rate
- Tight portal windows (typical ± 1-2s)
- Low false merge rate (<2%)
- Known people auto-identified without review

### Tools
- Portal statistics dashboard (timing distributions)
- Confidence calibration plots

---

## Phase 5: Polish (Months 5+)

### Goal
Nice-to-haves and advanced features.

### Tasks (if needed)
- License plate recognition (CodeProject.AI LPR)
- Re-ID optimization
- Identity clustering (group unknown faces)
- AOI/portal semantics (room-level tracking)

### Success Criteria
- System runs autonomously with minimal human review
- Complex scenarios handled (3+ people, 2+ vehicles)

---

## Key Metrics by Phase

| Phase | Auto-Merge Rate | Human Review/Day | False Merge Rate |
|-------|----------------|------------------|------------------|
| 0 | N/A | N/A | N/A |
| 1 | N/A | N/A | N/A |
| 2 (Week 5-6) | 0% | 20-30 | TBD |
| 2 (Week 7-8) | 30-50% | 10-20 | <5% |
| 3 | 60-70% | 5-10 | <3% |
| 4 | 90%+ | 1-2 | <2% |
| 5 | 95%+ | <1 | <1% |

---

## Risk Management

### High-Risk Items (Phase 2-3)
- ID swap (wrong merge breaks timeline)
- Face library pollution (wrong face registration)
- Multi-person merge disambiguation

**Mitigation:** Conservative thresholds, human validation before face registration

### Medium-Risk Items (Phase 4)
- Portal timing contamination (group behavior)
- LLM cost overrun

**Mitigation:** Solo track filter, cache LLM results

### Low-Risk Items (Phase 5)
- Noise promotion (tree shadow → agent)

**Mitigation:** Timeline review, manual cleanup

---

## Related Documents
- **L1 (Mission):** High-level success definition
- **L4 (Complexity):** What's hard and needs phasing
- **L11-13 (Technical):** Implementation tasks per phase
