# MarengoCam Planning Reminders

## Purpose
This file captures topics that need attention to prevent orphaning important issues. These are reminders, not a task list. They don't drive the next action but ensure nothing falls through the cracks during coherence work.

---

## Active Reminders

### L3_VideoIngestor.md - Stale RTSP Section
**Context:** File contains outdated RTSP streaming description (lines 58-83) contradicting Blue Iris JPEG acquisition model used elsewhere.

**Status:** Identified, not yet cleaned up

---

### L12 vs L13 Boundary
**Context:** L12_Merging.md contains detailed pseudocode. Unclear if this belongs in L12 (architecture/specs) or should move to L13 (implementation detail).

**Status:** Open question, needs decision on level boundaries

---

## Resolved

### Agent State Definitions (2025-01-13)
**Issue:** Inconsistent state definitions across L2, L10, L12 (person/vehicle states varied, `parked` vs `visible_parked`, missing `exited`, unclear `offscreen` vs `unknown`)

**Resolution:** Finalized state model with `visible`, `visible_in_vehicle`, `visible_parked`, `visible_moving`, `offscreen`, `in_structure(XYZ)`, `in_vehicle`, `unknown`, `exited`. Updated L2_Strategy.md, L10_Nomenclature.md, L12_Merging.md for consistency.

---

## Notes
- This file persists across all agent sessions working on MarengoCam planning
- Add reminders when coherence issues discovered, remove when resolved
- Keep entries concise - just enough context to recall the issue
