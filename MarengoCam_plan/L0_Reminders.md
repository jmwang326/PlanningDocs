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

### L3 Restructure to Match 7 Components (COMPLETED 2025-11-14)
**Issue:** L3 tactical docs should match the 7 components from L2_DecisionArchitecture exactly.

**Target structure (7 L3 files):**
1. L3_ChunkProcessing.md - ChunkProcessor tactics (203 lines) ✅
2. L3_TemporalLinking.md - TemporalLinker tactics (219 lines) ✅
3. L3_IdentityResolution.md - IdentityResolver tactics (360 lines, dense) ✅
4. L3_EvidenceProcessing.md - EvidenceProcessor tactics (354 lines, dense) ✅
5. L3_Gui.md - Timeline viewer, review queue (needs expansion) ⚠️
6. L3_Configuration.md - Portal config, thresholds (needs expansion) ⚠️
7. L3_SystemHealth.md - Performance monitoring (342 lines) ✅

**Archived files:** All old L3 and backup files moved to _archive/

**Status:** Core L3 files created, L3_Gui and L3_Configuration need expansion

---

### L4 Concept Library Structure (2025-11-14)
**Purpose:** L4 files provide detailed concept specifications that L3 references. L3 says "make an ID panel", L4 details what that means in words.

**Naming convention:** By feature - `L4_FeatureName.md`

**Cross-references:**
- L3 references L4 concepts: "See L4_IDPanel.md for detailed specification"
- L4 references L14 for implementation: "See L14_ReviewUI.py for implementation"
- L4 does NOT reference L13 (per user guidance)

**Created L4 files:**
- L4_ReviewPanel.md - Identity comparison panel specification ✅
- L4_ReviewQueue.md - Review queue priority and display ✅
- L4_HealthDashboard.md - System health dashboard ✅

**Planned L4 files (extracted from L3):**
- L4_GridLearning.md - 6×6 grid learning concept (from L3_IdentityResolution)
- L4_AlibiCheck.md - Alibi check concept (from L3_IdentityResolution)
- L4_PortalTransitions.md - Portal handling concept (from L3_Configuration, L3_IdentityResolution)
- L4_VehicleOccupancy.md - Vehicle occupancy tracking (from L12_VehicleOccupancy)
- L4_Timeline.md - Timeline viewer specification
- L4_ConfigurationUI.md - Configuration tool UI
- L4_FaceLibrary.md - Face library manager

**Status:** 3 files created, 7 planned for extraction

---

### File Structure Cleanup (COMPLETED 2025-11-14)
**Issue:** Root directory had many files; needed to organize D-files, old L-files, and extract L4 concepts.

**Completed actions:**
- ✅ Moved all D-files to `decisions/` subdirectory (6 files)
- ✅ Moved L4_Complexity.md to `decisions/` (risks/tradeoffs)
- ✅ Archived L2_Strategy_Principles.md, L15_Bootstrap.md
- ✅ Created 4 L4 concept files extracted from L3
- ✅ Archived all L12 files (8 files) - wrong abstraction level

**L12 files archived:**
- L12_Architecture.md, L12_ConcurrentTracking.md, L12_CrossCameraCorrelation.md
- L12_Merging.md, L12_ProcessOfElimination.md, L12_TrackAgentRelationship.md
- L12_VehicleOccupancy.md, L12_YOLOTrackingLeverage.md

**Why archived:** Too detailed (pseudocode belongs in L14), or obsolete architecture

**Status:** Cleanup complete

---

### L12 Redefinition (2025-11-14)
**Purpose:** L12 should be functional decomposition layer, not algorithm pseudocode.

**L12 = Interface contracts:**
- Function signatures (inputs/outputs)
- Call graphs (which functions call which)
- Data structures passed between functions
- Module organization
- NOT implementation pseudocode (that's L14)

**Example:**
```python
def resolve_identity(track: LocalTrack) -> IdentityResolution:
    """Calls: check_face_match(), check_grid_overlap(), perform_alibi_check()"""
```

**Current L12 files:** All archived (wrong abstraction level)

**Plan:** Generate new L12 files later when specifying function interfaces

**Status:** Clarified, ready for future implementation

---

### Prerequisites for Writing L12 (2025-11-14)
**Issue:** Before writing L12 (functional decomposition), we need foundational specs in place.

**L12 = Function signatures + call graphs + data flow**
- Cannot write `def resolve_identity(track: LocalTrack) -> IdentityResolution:` without knowing what `LocalTrack` is
- Cannot specify call graphs without understanding event model and orchestration

**Missing prerequisites:**

**Priority 1: L13_Database.md** (exists, needs review/update)
- Complete schema for LocalTrack, VehicleAgent, grid transitions
- Define all data types referenced by L12 function signatures
- Database tables, indexes, relationships

**Priority 2: L11_Data.md** (exists, needs review/update)
- Data flow architecture (polling vs events vs message queue)
- Event model (what triggers what)
- This unlocks call graphs for L12

**Priority 3: L13_Detection.md** (exists, needs review/update)
- LocalTrack data structure details (most-referenced type)
- Fields, types, relationships

**Priority 4: Handler orchestration**
- What's the main loop?
- What calls the 7 handlers and when?
- Maybe quick L12_Orchestration.md outline

**Status:** Blocked until L11/L13 specs complete. Come back to L12 later.

---
