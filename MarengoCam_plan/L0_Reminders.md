# MarengoCam Planning Reminders

## Purpose
This file captures topics that need attention to prevent orphaning important issues. These are reminders, not a task list. They don't drive the next action but ensure nothing falls through the cracks during coherence work.

---

## Active Reminders

### Hardcoded "10 cameras" Assumption
**Context:** L1, L4 documents assume "10 cameras" hardcoded (e.g., "10 cameras × 10 FPS = 100 FPS"). Should be configuration parameter.

**Locations:** L1_Mission.md:79, L1D_Mission.md:90-91, L4_Complexity.md:157

**Status:** Minor - should parameterize but not blocking

### Candidate Track Definition (RESOLVED 2025-11-14)
**Issue:** Undefined duration accumulation mechanism between motion detection and track creation.

**Resolution:** Introduced explicit `CandidateTrack` entity:
- Created when motion gate triggers (Standby → Armed)
- Accumulates detections during Armed state
- Promoted to LocalTrack if ≥3s sustained movement + movement_score ≥ threshold
- Discarded if motion stops or ARM timeout expires (ephemeral, not persisted)
- Details in L13_TrackStateManager.md, referenced from L3_DetectionAndTracking.md

**Key insight:** Candidate track lifetime = Armed state. Decouples motion detection from agent creation.

**Changes:** L10_Nomenclature, L3_DetectionAndTracking, L13_TrackStateManager (new content), L2_Strategy (face library safety), L4_Complexity (candidate risks), L13_Detection (cross-reference)

---

### Track-Centric Model: Standby/Armed Track Initiation (NEEDS SPEC 2025-11-14)
**Issue:** With track-centric architecture (LocalTrack as fundamental unit), need to clarify how tracks start from Standby/Armed states.

**Questions:**
- How does motion gate trigger relate to LocalTrack creation?
- Is "candidate track" an implementation detail or part of LocalTrack lifecycle?
- When does ChunkProcessor create a LocalTrack vs discard detections?
- How does ≥3s movement threshold fit with 12s chunks?

**Status:** Reminder to spec this out in L3_ChunkProcessing or L13_Detection after core architecture stabilizes

**Related:** L2_DecisionArchitecture (ChunkProcessor), L10_Nomenclature (deprecated "Candidate Track")

---

### L12 vs L13 Boundary
**Context:** L12_Merging.md contains detailed pseudocode. Unclear if this belongs in L12 (architecture/specs) or should move to L13 (implementation detail).

**Status:** Open question, needs decision on level boundaries

---

## Notes
- This file persists across all agent sessions working on MarengoCam planning
- Add reminders when coherence issues discovered, remove when resolved
- Keep entries concise - just enough context to recall the issue

### L3 Tactical Docs Need Handler Alignment (2025-11-14)
**Issue:** L3 tactical documents use old terminology and component names, need alignment with track-centric architecture and 4 handlers.

**Mapping:**
- L3_DetectionAndTracking.md → ChunkProcessor + TemporalLinker tactics
- L3_TrackMerging.md → IdentityResolver tactics (rename needed)
- L3_EvidenceEngine.md → Part of IdentityResolver (sub-component)
- L3_TrackStateManager.md → Orchestration (currently stub)

**Terminology updates needed:**
- "Track segment" → "LocalTrack"
- "Track Merging Engine" → "IdentityResolver"
- Add handler cross-references to L2_DecisionArchitecture

**Status:** In progress - L2 level updated (L2_Strategy, L2_DecisionArchitecture, L10_Nomenclature), L3 updates pending

---

### L3 Restructure to Match 7 Components (2025-11-14)
**Issue:** L3 tactical docs should match the 7 components from L2_DecisionArchitecture exactly.

**Target structure (7 L3 files, ~100 lines each):**
1. L3_ChunkProcessing.md - ChunkProcessor tactics (filtering, motion gate, ≥3s threshold)
2. L3_TemporalLinking.md - TemporalLinker tactics (overlap zone, appearance similarity)
3. L3_IdentityResolution.md - IdentityResolver tactics (evidence hierarchy, grid learning, alibi checks, process of elimination) - **may be ~150-200 lines (dense but acceptable)**
4. L3_EvidenceProcessing.md - EvidenceProcessor tactics (uncertainty resolution, cascading updates)
5. L3_Gui.md - Timeline viewer, review queue (exists)
6. L3_Configuration.md - Portal config, thresholds (exists)
7. L3_SystemHealth.md - Performance monitoring (NEW)

**Files to consolidate/move:**
- L3_DetectionAndTracking.md → split to L3_ChunkProcessing + L3_TemporalLinking
- L3_TrackMerging.md → rename to L3_IdentityResolution.md
- L3_EvidenceEngine.md → merge into L3_IdentityResolution or L3_EvidenceProcessing
- L3_HumanInTheLoop.md → merge into L3_EvidenceProcessing
- L3_InferenceManager.md, L3_VideoIngestor.md, L3_PanelAssemblyService.md, L3_TimelineReconstruction.md, L3_TrackStateManager.md → move to L13 or _archive

**Detailed algorithms push to L12:**
- L12_GridLearning.md (if L3_IdentityResolution gets too long)
- L12_AlibiCheck.md
- L12_ProcessOfElimination.md (already exists)
- L12_Merging.md (already exists)

**Status:** L2 streamlined (complete), L3 restructure pending

---
