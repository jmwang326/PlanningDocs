# MarengoCam Planning Reminders

## Purpose
This file captures topics that need attention to prevent orphaning important issues. These are reminders, not a task list. They don't drive the next action but ensure nothing falls through the cracks during coherence work.

---

## Active Reminders

### Hardcoded "10 cameras" Assumption
**Context:** L1, L4 documents assume "10 cameras" hardcoded (e.g., "10 cameras × 10 FPS = 100 FPS"). Should be configuration parameter.

**Locations:** L1_Mission.md:79, L1D_Mission.md:90-91, L4_Complexity.md:157

**Status:** Minor - should parameterize but not blocking

### L12 vs L13 Boundary
**Context:** L12_Merging.md contains detailed pseudocode. Unclear if this belongs in L12 (architecture/specs) or should move to L13 (implementation detail).

**Status:** Open question, needs decision on level boundaries

---

## Resolved

### L3 Acquisition & Detection Coherence (2025-01-13)
**Issues:**
1. Conflicting 7-10s pre-roll buffer vs 25s processing buffer
2. Chicken-and-egg: sparse 10% YOLO sampling in Standby might miss detections
3. Neighbor boost complexity (adjacency tracking via 8x8 grid)
4. Confusing chunk overlap math

**Resolution:**
1. Removed pre-roll buffer (stale - not needed for large property with fast detection)
2. Clarified motion gate runs BEFORE sparse YOLO (all frames → motion gate → if motion → sparse YOLO 10%)
3. Removed neighbor boost entirely (stale - all cameras polled continuously, each detects independently)
4. Fixed chunk math explanation (12s chunk with 2s overlap, extracted every 10s)

**Key insight:** All cameras polled at 10 FPS constantly. Camera states (Standby/Armed/Active/Post) control inference rate and frame persistence, not acquisition.

### Agent State Definitions (2025-01-13)
**Issue:** Inconsistent state definitions across L2, L10, L12 (person/vehicle states varied, `parked` vs `visible_parked`, missing `exited`, unclear `offscreen` vs `unknown`)

**Resolution:** Finalized state model with `visible`, `visible_in_vehicle`, `visible_parked`, `visible_moving`, `offscreen`, `in_structure(XYZ)`, `in_vehicle`, `unknown`, `exited`. Updated L2_Strategy.md, L10_Nomenclature.md, L12_Merging.md for consistency.

### Pre-Roll References (2025-01-13)
**Issue:** Multiple documents referenced pre-roll buffer (7-10s) that was removed. Conflated acquisition rate (constant 10 FPS) with inference rate (variable by state).

**Resolution:**
- Removed L3_Acquisition.md (stale, superseded by L3_VideoIngestor.md and L3_DetectionAndTracking.md)
- Updated L10_Nomenclature.md to remove pre-roll definition, clarify 25s buffer provides historical frames as artifact
- Updated L1_Mission.md, L12_Architecture.md to reference 25s processing buffer instead of pre-roll
- Updated L11_Acquisition.md to clarify constant 10 FPS acquisition, 25s rolling buffer, inference rates separate
- Updated README.md to reference current L3 files

---

## Notes
- This file persists across all agent sessions working on MarengoCam planning
- Add reminders when coherence issues discovered, remove when resolved
- Keep entries concise - just enough context to recall the issue
