# Level 2D - Strategy Discourse

## Why Evidence Hierarchy (Not Weighted Formula)?

**Decision:** Use evidence hierarchy (strongest first) instead of weighted scoring.

**Rationale:**
- Face match alone is definitive (no need to check other evidence)
- Time/space gating eliminates impossible candidates before expensive LLM
- Cheaper/faster methods first, expensive fallback

**Tradeoff:**
- Miss cases where weak evidence combined would be strong
- But: simpler, faster, more debuggable

---

## Why Accept Uncertainty?

**Decision:** Allow tracks to remain unassigned or multi-assigned.

**Rationale:**
- Wrong merge breaks timeline (hard to fix)
- Uncertain merge preserves options (can resolve later)
- Forensic system (not real-time), uncertainty acceptable

**Tradeoff:**
- Timeline has gaps or ambiguities
- But: better than false positives

---

## Why Portal Configuration (Not Learning)?

**Decision:** User manually configures portals (camera adjacencies).

**Rationale:**
- User knows where doors/gates are (ground truth)
- Statistical discovery of portals wastes time
- Portal configuration narrows merge candidates (only check connected cameras)

**Tradeoff:**
- Requires manual setup
- But: one-time cost, prevents wrong adjacencies

---

## Why Not Portal Timing Learning?

**Decision:** Portal timing does NOT help merge decisions.

**Rationale:**
- Initial idea: learn "typical_time" through portal, tighten windows to reject false candidates
- Reality: timing variance too high (people pause, groups move slow, rushing vs walking)
- Face recognition + time/space gating already sufficient
- Tight timing windows would reject legitimate merges

**What we DO use portals for:**
- Camera adjacency (which cameras connect)
- Candidate narrowing (only check connected cameras, not all pairs)

**What we DON'T use portals for:**
- Timing-based rejection ("too slow through portal")
- Confidence scoring based on timing fit

---

## Why Face Recognition Before Re-ID?

**Decision:** Prioritize face recognition in evidence hierarchy.

**Rationale:**
- Already available, mature, trainable
- Handles most merge cases (people have faces)
- Re-ID useful for vehicles, deferred

**Tradeoff:**
- Vehicles require LLM until Re-ID available
- But: faster automation path

---

## Why No Probabilities for Multi-Assignment?

**Decision:** Track marked as "possibly Agent A or B" (no percentages like 60%/40%).

**Rationale:**
- Probabilities hard to justify (where does 60% come from?)
- Binary possibilities simpler to reason about
- Timeline UI shows "or" not "60% likely"

**Tradeoff:**
- Can't express relative confidence
- But: avoids false precision, clearer to user

---

## Note: Decisions vs Discourse

**Clarification:** "Decisions" and "Discourse" are essentially the same thing.
- D-files document decisions made during discourse
- They capture rationale, tradeoffs, alternatives considered
- Brief record of "why we chose this approach"

---

## Why 6×6 Inter-Camera Grid?

**Decision:** Use 6×6 grid per camera for learning spatial adjacency.

**Rationale:**
- **Overlap detection**: Grid learns which camera regions overlap (typical_time ≈ 0)
- **Auto-merge**: Single agent in overlap zone → must be same agent, auto-merge
- **Portal learning**: Grid learns portal connections without manual configuration
- **Tractability**: 36 cells × 36 cells = 1,296 edges per camera pair (sparse in practice)
- **Resolution**: Fine enough to distinguish "left vs right side of driveway"

**Why 6×6 vs 8×8:**
- Original 8×8 was 4:1 downsample from 32×32 intra-camera grid
- New framework removes 32×32 (YOLO11 provides tracking directly)
- 6×6 simpler to visualize, still effective for spatial correlation

**What grid does:**
1. **Overlap zones**: Auto-merge single agents (no LLM needed)
2. **Portal zones**: Narrow time windows, filter candidates
3. **No-connection zones**: Reject merge (cameras not adjacent)

**What grid does NOT:**
- Does NOT replace face recognition (face still strongest evidence)
- Does NOT enforce strict timing (variance accommodated)
- Does NOT require manual portal clicking (learns from validated merges)

**Tradeoff:**
- Requires bootstrap phase (historical footage processing)
- But: massive reduction in LLM calls for overlapping cameras
- Grid auto-merge >> manual review for simple cases

---

## Change Log

**2025-11-13**:
- Initial strategy document created during framework reorganization
- Added portal timing decision (doesn't help merge decisions)
- Added multi-assignment probabilities decision (no percentages)
- Added 6×6 grid rationale (overlap detection, auto-merge)
