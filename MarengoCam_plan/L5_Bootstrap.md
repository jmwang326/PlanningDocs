# Level 5 - Bootstrap Strategy

## Purpose
Accelerate system learning by building face library and Re-ID training sets **concurrently** with Phase 1 deployment. This is not a blocking prerequisite - it runs in parallel as a manual workflow using dedicated GUI tools.

**Key insight:** Don't wait for the system to be "smart" before deploying. Deploy basic system, then use GUI tools to rapidly build training data from live footage.

---

## Bootstrap Goals

### 1. Face Library (Priority 1)
**Objective:** Identify 5-10 most important people (family, regular visitors) to enable auto-merge.

**Why critical:** Face recognition (≥ 0.75 confidence) is the strongest evidence for auto-merge. Even 5 known faces dramatically reduces review queue.

**Target:** 50-100 face crops per person, covering:
- Multiple cameras (cross-camera variation)
- Multiple lighting conditions (day/night, indoor/outdoor)
- Multiple angles (frontal, profile)

### 2. Re-ID Training Set (Priority 2)
**Objective:** Collect training data for future Re-ID model fine-tuning.

**Why needed:** Generic Re-ID models don't understand this specific environment. Custom training improves appearance-based matching.

**Target:**
- **People:** 20-30 validated agents × 50+ crops each = 1000-1500 person crops
- **Vehicles:** 10-15 validated vehicles × 30+ crops each = 300-500 vehicle crops

**Diversity required:**
- Cross-camera (same person/vehicle on different cameras)
- Temporal variation (same person, different clothing/lighting)
- Quality variation (sharp focus + motion blur samples)

### 3. Grid Learning (Automatic)
**Objective:** Populate 6×6 grid with travel times from validated merges.

**Why low priority:** Grid learns automatically from any validated merge (face match or human review). No manual work needed.

**Target:** 80% coverage of common paths within 2-4 weeks.

---

## Concurrent Workflow: Phase 1 + Bootstrap

**Phase 1 runs:** ChunkProcessor + TemporalLinker + IdentityResolver + Review Queue
**Bootstrap runs:** Human reviews uncertain tracks, builds face library, collects Re-ID crops

```
Day 1-7:
  System: Creates LocalTracks, queues everything for review (no grid, no faces)
  Human: Reviews 50-100 tracks/day, identifies 5-10 key people
         Adds faces to library using Face Manager GUI

Day 8-14:
  System: Auto-merges tracks with face match (≥ 0.75)
         Still queues uncertain tracks
  Human: Reviews remaining uncertain tracks
         Continues adding face library samples (different lighting, angles)
         Exports Re-ID crops for future training

Day 15-30:
  System: 60-80% auto-merge rate (face + grid learning)
  Human: Reviews only complex cases (groups, occlusions)
         Curates Re-ID training set quality
```

**Result:** System learns while running. Human effort decreases exponentially (100/day → 20/day → 5/day).

---

## GUI Tools (Separate Applications)

### 1. Review Queue Interface
**Purpose:** Review uncertain tracks, make merge decisions

**Workflow:**
1. See prioritized queue (most recent, simplest first)
2. Open comparison panel (uncertain track vs candidates)
3. Decide: merge / reject / create new agent
4. Optional: Tag for face library or Re-ID export

**Implementation:** Web UI (L3_Gui.md, L4_ReviewQueue.md, L4_ReviewPanel.md)

**Usage:** Daily during bootstrap (decreases over time)

---

### 2. Face Library Manager
**Purpose:** Build and curate face recognition training library

**Workflow:**
1. Browse all face crops (grouped by agent)
2. Assign names to agents ("John Doe", "Delivery Driver")
3. Add/remove face samples per agent
4. View coverage: cameras × lighting conditions
5. Trigger face library retrain (CodeProject.AI)

**Implementation:** Standalone web UI (L4_FaceLibrary.md - TBD)

**Usage:** Heavy during Days 1-14, occasional afterward (new people)

---

### 3. Re-ID Crop Exporter
**Purpose:** Export validated tracks as labeled Re-ID training set

**Workflow:**
1. Filter tracks: validated merges only, high quality
2. Select agents with good cross-camera coverage
3. Export crops per agent (folder structure for training)
4. Verify diversity (cameras, lighting, angles)
5. Generate training manifest (agent_id, camera, timestamp, path)

**Implementation:** CLI tool or simple web UI

**Usage:** Batch export every 1-2 weeks during bootstrap

**Output format:**
```
reid_dataset/
  person_123/
    camera_A_day_001.jpg
    camera_A_night_002.jpg
    camera_B_day_003.jpg
    ...
  person_456/
    ...
  vehicle_789/
    ...
  manifest.json  # Training metadata
```

---

### 4. Grid Visualization Tool
**Purpose:** Monitor grid learning progress

**Workflow:**
1. Select camera pair (Camera_A → Camera_B)
2. View 6×6 heatmap (learned travel times)
3. Identify gaps (unlearned paths)
4. Identify overlap zones (< 0.4s)

**Implementation:** Desktop UI (tkinter) or web UI

**Usage:** Occasional (debug grid learning, verify coverage)

---

## Bootstrap Metrics

### Face Library Progress
- **Agent count:** How many people identified
- **Coverage per agent:** Cameras × lighting (target: 3 cameras × 2 lighting = 6 variants)
- **Crop count per agent:** Target 50-100
- **Quality distribution:** Sharp vs blurred

### Re-ID Dataset Progress
- **Agent count:** Unique people/vehicles with ≥ 50 crops
- **Cross-camera coverage:** % of agents with ≥ 2 cameras
- **Temporal diversity:** % of agents with day + night samples
- **Total crops:** Target 1000+ people, 300+ vehicles

### Grid Learning Progress
- **Path coverage:** % of common camera pairs with learned times
- **Overlap zones detected:** Count of < 0.4s transitions
- **Auto-merge rate:** % of tracks merged without review

**Target:** 80% auto-merge rate by Day 30

---

## Phase Transition: Online Learning → Autonomous

**Online Learning Mode (Days 1-30):**
- High face threshold (0.85)
- All uncertain tracks queued for review
- Human validates all face library additions
- Conservative grid learning (high-confidence only)

**Autonomous Mode (Day 30+):**
- Lower face threshold (0.70)
- Auto-merge enabled (high-confidence decisions)
- LLM reviews most uncertain tracks
- Human reviews only complex cases

**Transition criteria:**
- Auto-merge rate > 80%
- Face library > 5 key people
- Grid learning > 80% common paths

**See:** L3_Configuration.md for mode configuration

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - 7 components (ChunkProcessor, TemporalLinker, IdentityResolver, EvidenceProcessor)
- **L2_Strategy.md** - Learn from clean data, human-in-loop initially

### Tactics
- **L3_IdentityResolution.md** - Grid learning, face library, evidence hierarchy
- **L3_EvidenceProcessing.md** - Review queue management
- **L3_Gui.md** - GUI overview (separate applications)
- **L3_Configuration.md** - Online learning vs autonomous mode
- **L3_SystemHealth.md** - Bootstrap phase metrics

### L4 Concepts
- **L4_ReviewPanel.md** - Review queue interface spec
- **L4_GridLearning.md** - Grid learning concept
- **L4_FaceLibrary.md** (planned) - Face library manager spec

### Deployment
- **L6_Deployment.md** - Phased deployment plan
