# MarengoCam Documentation

This directory contains the complete architectural specification for MarengoCam, a multi-camera surveillance system with agent tracking and cross-camera merging.

---

## � STARTING IMPLEMENTATION?

**Read these first (in project root, not docs/):**
1. **`../SETUP.md`** - Step-by-step setup instructions ⭐⭐⭐
2. **`../PROGRESS.md`** - Current implementation status ⭐⭐
3. **`../IMPLEMENTATION_LOG.md`** - Session-by-session history ⭐

**Then come back here for architecture details.**

---

## �📖 Start Here

### **MASTER_ROLLOUT.md** - Complete Implementation Guide ⭐⭐⭐
**100% redundant compilation** of architecture, implementation, and rollout sequencing. Single reference for understanding what to build and when.

**Contains:**
- Architecture summary (agent-centric tracking, dual-grid, no motion gate)
- System components (acquisition, detection, tracking, merging)
- Phase-by-phase rollout (Phase 0-5 with detailed tasks)
- Implementation order (what to build, in what order)
- Key design principles (codifiable/observable, configuration over learning)
- Why this ordering (data capture first, prove one thing at a time)

**Read if:** You want a complete end-to-end understanding or need to start implementation.

---

### **NOMENCLATURE.md** - Standard Terminology Reference ⭐
The definitive guide to terminology used throughout the project.

**Contains:**
- Core entities (agent, track segment, detection)
- Grid systems (32×32 intra-camera, 8×8 inter-camera)
- Camera states (Standby, Armed, Active, Post, Suppressed)
- Merge decision hierarchy (Auto-Merge, Queue, Reject, CANNOT)
- Configuration terms (Stage vs Phase)
- Formatting conventions and common mistakes

**Read if:** You're unsure about terminology or need to check naming conventions.

---

### **BIGPICTURE_V2.md** - High-Level Architecture (Source of Truth)
The authoritative overview of the entire system. Read this first (or start with MASTER_ROLLOUT.md for complete guide).

**Contains:**
- Core philosophy and goals
- Agent-centric tracking model
- Dual-grid system (32×32 intra, 8×8 inter)
- Merge decision hierarchy
- Storage and retention policies
- References to all other docs

**Read if:** You're new to the project or need a 10,000-foot view.

---

## 🔧 Implementation Guides

### **MASTER_ROLLOUT.md** - Complete Implementation Roadmap ⭐⭐⭐
**100% redundant compilation** of architecture, implementation, and rollout sequencing.

**Read if:** You want end-to-end understanding or need to start implementation.

---

### **COMPLEXITY_AND_TUNING_RISKS.md** - Critical Review Document ⭐⭐
Identifies all decision logic that is complicated (failure-prone) or tuning-dependent (sensitive to threshold changes).

**Contains:**
- Complexity analysis of merge logic (time/space filtering, Re-ID disambiguation, multi-person handling)
- Portal timing statistics risk assessment
- Face recognition and LLM adjudication complexity
- Risk levels (HIGH/MODERATE/LOW) for each component
- Recommended testing strategy and tuning priorities
- Decision on what to document vs what to monitor during rollout

**Read if:** You're concerned about failure modes, need to prioritize testing, or want to understand which areas require empirical calibration vs which are architecturally sound.

---

### **DATA_AND_STORAGE_ARCHITECTURE.md** - Complete Data Pre-Planning ⭐⭐⭐
Comprehensive pre-planning of all data structures, storage formats, functions, and libraries needed across all phases.

**Contains:**
- Data categories (learned data, raw data, metadata, configuration)
- Directory structure (config, state, storage, metadata, logs, tools)
- Learned data schemas (height grids, portal stats, **identity registry**, face library, Re-ID)
- Raw data formats (JPEG frames, face crops, panels, MP4 clips)
- Metadata database schema (SQLite: agents, tracks, detections, merges, validations)
  - **Multi-scale grid tracking** (32×32 fine grid + 8×8 coarse grid with rationale)
  - **Solo track metadata** (for clean portal timing learning)
  - **Merge complexity analysis** (O(1) guarantees, computational efficiency)
- Configuration files (cameras, portals, thresholds, services, stages)
- Function inventory by phase (what functions to implement when)
- **Identity management** (central registry prevents duplicate people/vehicles)
- Library dependencies (all dependencies locked in)
- Configuration UI tools (portal editor, camera params viewer, threshold tuner, **identity picker**)
- Data migration & versioning strategy
- Pre-positioned data fields (bootstrap with nullables for later phases)

**Read if:** You're implementing data storage, designing database schema, building configuration tools, or need to understand how data flows through the system.

---

### **HARD_VS_FIGURE_OUT.md** - Master Task Checklist
Complete task breakdown with phase tags [Phase 0-5].

**Contains:**
- Hard problems (tuning/calibration)
- Figure-out tasks (deterministic implementation)
- Test criteria for each task
- Phase-by-phase rollout guidance

**Read if:** You're implementing features or planning rollout.

---

### **AGENT_TRACKING_AND_MERGE.md** - Tracking & Merge Logic (1078 lines)
Comprehensive specification for **agent persistence and cross-camera merging algorithms**.

**Contains:**
- Dual-grid mechanics (32×32 → 8×8 downsampling)
- Agent data structures and persistence model
- Merge decision pipeline (4 phases: time/space, face, evidence, LLM)
- Re-ID per camera architecture (Phase 4-5 deferred)
- Image bank and quality metrics
- Vehicle containers and multi-state tracking

**Read if:** Implementing tracking algorithms, merge logic, or Re-ID systems.

**Complements EVENT_INFERENCE_SPEC.md:** This doc focuses on *what to track and how to merge*, EVENT_INFERENCE focuses on *how to acquire frames and schedule inference*.

---

### **EVENT_INFERENCE_SPEC.md** - Inference & Archiving Mechanics
Detailed specification for **pipeline plumbing and infrastructure**.

**Contains:**
- Frame acquisition (HTTP transport, retry logic, main/sub selection)
- Motion gate (prefilter to save GPU budget)
- Inference scheduler (batch sizing, backpressure, GPU queue management)
- Camera state machine (Standby/Armed/Active/Post transitions)
- Ring buffers and manifest generation
- Storage paths and retention
- Face pipeline (crop extraction)
- Grid system details (Section 14 needs update to 32×32)

**Read if:** Implementing acquisition pipeline, storage systems, or state machine.

**Complements AGENT_TRACKING_AND_MERGE.md:** This doc focuses on *how to get frames and detect objects*, AGENT_TRACKING focuses on *what to do with detections*.

---

### **PORTAL_CONFIGURATION.md** - Camera Connections

Manual portal configuration for cross-camera tracking.

**Contains:**
- Portal definition (doors, gates, exits)
- Configuration format (YAML)
- No learning, no statistics
- Simple adjacency filter

**Read if:** Configuring property layout or implementing cross-camera merge logic.

**Complements AGENT_TRACKING_AND_MERGE.md:** This doc defines *which cameras connect*, AGENT_TRACKING uses portals for merge candidates.

---

## 📚 Reference Documents

### **ARCHITECTURE_UPDATES_NOV2025.md** - Executive Summary
November 2025 architectural decisions (12 core decisions).

**Contains:**
- Dual-grid rationale (why 32×32 and 8×8)
- Agent persistence model (why agent-centric vs detection-centric)
- Merge hierarchy (why face-first)
- Face recognition priority (why CodeProject.AI before Re-ID)
- Class-agnostic edges (why same edges for all agents)
- Observable evidence only (why no statistical priors)

**Read if:** You need context on *why* we made these decisions, or briefing stakeholders.

**Complements all other docs:** This provides the *reasoning*, other docs provide the *implementation*.

---

### **EVENT_INFERENCE_SPEC.md** - Inference & Archiving Mechanics
Detailed inference pipeline, tracking, and storage mechanics.

**Contains:**
- Frame acquisition and ring buffers
- Camera state machine (Standby/Armed/Active/Post)
- Detection and tracking flow
- Manifest generation
- Face pipeline (crop extraction)
- Grid system details (Section 14 needs update to 32×32)

**Read if:** Implementing acquisition, detection, or storage systems.

---

## ⚙️ Configuration & Operations

### **CONFIGURATION_SYSTEM.md** - Runtime Configuration
Complete configuration architecture including stages, external AI services, and resource throttling.

**Contains:**
- Operational stages (DEV_SLOW, DEV_FULL, PROD_LEARNING, PROD_AUTO)
- External AI service registry (multi-CodeProject.AI support)
- Service health tracking and circuit breakers
- Resource throttling (adaptive frame rate)
- Hot-reload and secrets management
- Config file format (YAML)

**Read if:** Setting up system configuration, managing services, or deploying to different hardware.

---

### **UI_ARCHITECTURE.md** - User Interface Design
Separate, focused applications per phase (config, viewing, training, timeline).

**Contains:**
- UI apps by phase (0-5)
- Desktop apps (tkinter): Camera Grid Viewer, Single-Camera Debug
- Web apps (Flask): Merge Queue, Portal Config, Face Library, Timeline
- CLI tools (Click): Health monitor, config management, secrets
- API design for web UIs
- Consolidation strategy (Phase 5+)

**Read if:** Building user interfaces or planning review workflows.

---

## 📚 Reference Documents

### **ARCHITECTURE_UPDATES_NOV2025.md** - Executive Summary
November 2025 architectural decisions (12 core decisions).

**Contains:**
- Dual-grid rationale
- Agent persistence model
- Merge hierarchy
- Face recognition priority
- Class-agnostic edges
- Observable evidence only

**Read if:** You need a quick summary of key architectural decisions for stakeholders.

---

## 🗂️ Document Dependencies

```
NOMENCLATURE.md (terminology reference)
    │
BIGPICTURE_V2.md (start here)
    │
    ├─→ CONFIGURATION_SYSTEM.md (stages, services, config)
    ├─→ UI_ARCHITECTURE.md (UIs per phase)
    │
    ├─→ HARD_VS_FIGURE_OUT.md (task checklist)
    │       │
    │       ├─→ AGENT_TRACKING_AND_MERGE.md (tracking/merge details)
    │       ├─→ PORTAL_CONFIGURATION.md (camera connections)
    │       └─→ EVENT_INFERENCE_SPEC.md (inference/storage details)
    │
    └─→ ARCHITECTURE_UPDATES_NOV2025.md (executive summary)
```

---

## 🎯 Quick Reference by Task

| I want to... | Read this document |
|--------------|-------------------|
| **Check terminology** | `NOMENCLATURE.md` ⭐ |
| Understand the overall system | `BIGPICTURE_V2.md` |
| **Understand complexity/risks** | `COMPLEXITY_AND_TUNING_RISKS.md` ⭐⭐ |
| Know what to build next | `HARD_VS_FIGURE_OUT.md` |
| Implement tracking/merging | `AGENT_TRACKING_AND_MERGE.md` |
| Configure portals | `PORTAL_CONFIGURATION.md` |
| Implement acquisition/storage | `EVENT_INFERENCE_SPEC.md` |
| Configure stages/services | `CONFIGURATION_SYSTEM.md` |
| Build user interfaces | `UI_ARCHITECTURE.md` |
| Brief a stakeholder | `ARCHITECTURE_UPDATES_NOV2025.md` |
| **Design data schemas** | `DATA_AND_STORAGE_ARCHITECTURE.md` ⭐⭐⭐ |

---

## 🚀 Recommended Reading Order

### **First Time (Getting Oriented):**
1. `NOMENCLATURE.md` - Learn the terminology
2. `BIGPICTURE_V2.md` - Understand the vision
3. `ARCHITECTURE_UPDATES_NOV2025.md` - Key decisions
4. `HARD_VS_FIGURE_OUT.md` - See the roadmap

### **Phase 0 Implementation:**
1. `CONFIGURATION_SYSTEM.md` - Set up stages and config
2. `UI_ARCHITECTURE.md` - Build Camera Grid Viewer + CLI tools
3. `EVENT_INFERENCE_SPEC.md` - Build acquisition pipeline
4. `HARD_VS_FIGURE_OUT.md` [Phase 0 tasks] - Task checklist

### **Phase 1 Implementation:**
1. `AGENT_TRACKING_AND_MERGE.md` (intra-camera sections) - Build tracking
2. `UI_ARCHITECTURE.md` (Single-Camera Debug Viewer) - Build debug UI
3. `HARD_VS_FIGURE_OUT.md` [Phase 1 tasks] - Task checklist

### **Phase 2 Implementation:**
1. `AGENT_TRACKING_AND_MERGE.md` (merge sections) - Build merging
2. `PORTAL_CONFIGURATION.md` - Define camera connections
3. `CONFIGURATION_SYSTEM.md` (External AI Services) - Set up CodeProject.AI
4. `UI_ARCHITECTURE.md` (Merge Queue) - Build review UI
5. `HARD_VS_FIGURE_OUT.md` [Phase 2 tasks] - Task checklist

---

## 📝 Document Maintenance

**Last Updated:** November 7, 2025

**Change Log:**
- Removed stale Phase 1 MQTT docs (conflicted with JPEG-only plan)
- Consolidated CONFIG_AND_LEARNING_UI.md into UI_ARCHITECTURE.md
- Removed DATA_RETENTION_AND_STORAGE.md (covered in BIGPICTURE_V2.md + CONFIGURATION_SYSTEM.md)
- Removed EXTERNAL_AI_INTEGRATIONS.md (superseded by CONFIGURATION_SYSTEM.md)

**Known Issues:**
- Old terminology in some code comments (update during refactoring)

---

## 🆘 Getting Help

If documentation is unclear:
1. Check `NOMENCLATURE.md` for terminology
2. Check `BIGPICTURE_V2.md` for high-level context
3. Refer to the specific implementation doc for details
4. Cross-reference with `HARD_VS_FIGURE_OUT.md` for test criteria
5. Update docs if you find gaps or ambiguities (keep them alive!)

---

**Happy building! 🎉**
