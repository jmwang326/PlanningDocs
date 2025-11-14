# Level 2 - Strategy

## Purpose
Non-technical description of **what** we're doing and **why** this approach solves the problem.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for the technical architecture (7 components: ChunkProcessor, TemporalLinker, IdentityResolver, EvidenceProcessor, GUI, Configuration, SystemHealth).

---

## Core Strategy: Manage Video Clips (Tracks)

**The fundamental insight:** Multi-camera surveillance is a **clip management problem**.

Every 10 seconds, YOLO processes 12-second video chunks and produces observations (LocalTracks). Our job is to:
1. **Link clips** from the same person across time (same camera)
2. **Identify clips** as belonging to specific people (cross-camera)
3. **Handle uncertainty** when we can't be sure

Everything else - timelines, alerts, forensics - is just querying and displaying these clips.

---

## Strategic Principles

### 1. Filter Noise First
**Why:** Trees, shadows, brief glitches create false detections.

**Approach:**
- Motion gate filters static frames
- Only sustained movement (≥3s) creates tracks
- Only high-confidence detections persist

**Result:** Clean data from the start.

**See:** L3_ChunkProcessing.md for filtering tactics

---

### 2. Link Tracks Over Time
**Why:** YOLO assigns new IDs each chunk. We need continuous observation on one camera.

**Approach:**
- 2s overlap between chunks
- Match by position + appearance in overlap zone
- Build linked list of tracks (previous/next pointers)

**Result:** Continuous timeline on one camera, even though YOLO IDs change.

**See:** L3_TemporalLinking.md for linking tactics

---

### 3. Merge Across Cameras Using Evidence Hierarchy
**Why:** Timeline requires knowing when Track A (Camera 1) = Track B (Camera 2) = same person.

**Evidence Hierarchy:**
1. **Face Recognition** (definitive) → auto-merge
2. **Grid Overlap** (< 0.4s travel, single agent) → auto-merge
3. **Grid Transition + Alibi Check** (fits learned path, no alternatives) → auto-merge
4. **LLM / Human Adjudication** (ambiguous) → queue for review
5. **Reject** (face mismatch, impossible timing)

**Why this order:** Strongest evidence first, fallback to weaker evidence.

**See:** L3_IdentityResolution.md for merging tactics (grid learning, alibi checks, process of elimination)

---

### 4. Accept Uncertainty
**Why:** Evidence may be missing (face occluded, timing ambiguous, multiple candidates).

**Approach:**
- Don't force wrong merge to avoid uncertainty
- Mark track with `identity_candidates` set
- Uncertainty persists until evidence resolves it
- System operates with partial timeline

**Result:** Never force wrong decisions. Wait for evidence.

**See:** L3_EvidenceProcessing.md for uncertainty resolution tactics

---

### 5. Learn from Clean Data
**Why:** Grid learning and face library improve over time, but only if trained on clean data.

**What gets learned:**
- **Grid:** Travel times between cameras (from validated merges, single agent only)
- **Face library:** Face recognition (from high-confidence merges ≥ 0.75 or validated human review)

**What gets excluded:**
- Groups (multiple people together)
- Ambiguous merges (uncertain identity)
- Sporadic detections (person in vehicle)
- Low-quality tracks (partial visibility)

**Result:** System gets smarter without cascading errors.

**See:** L3_IdentityResolution.md for grid learning tactics

---

### 6. No Probability Scores
**Why:** Can't compute accurate weights (0.3×time + 0.5×face = nonsense). Can't debug ("why 0.65?").

**Approach:**
- `identity_candidates` is a **set** (not probabilities)
- Evidence is countable and explicit
- Process of elimination removes impossible candidates

**Result:** Debuggable, transparent decisions.

**See:** L3_EvidenceProcessing.md for process of elimination tactics

---

## Special Cases

### Concurrent Agents (Same Camera)
**Strategy:** YOLO maintains separate tracking IDs (different local_ids), so no special handling needed. Exclude groups from grid learning.

**See:** L12_ConcurrentTracking.md for details

---

### Vehicle Occupancy *(NEEDS MORE SPEC)*
**Strategy:** Mark person track with `inside_vehicle` context flag. Use sporadic detections for face matching only. Accept gaps in person timeline (vehicle track becomes proxy).

**See:** L3_VehicleOccupancy.md for detailed strategy (TBD)

---

### Structure Entry/Exit *(NEEDS MORE SPEC)*
**Strategy:** Mark track with `at_portal` context flag. Accept location uncertainty (agent in structure). Use face match or process of elimination to resolve when agent exits.

**See:** L3_StructureEntry.md for detailed strategy (TBD)

---

### Property Boundaries *(NEEDS MORE SPEC)*
**Strategy:** Mark with `property_entry` context flag. Vehicle arriving from street → occupancy uncertain (could have hidden people). Accept uncertainty until people exit vehicle on property.

**See:** L3_PropertyBoundaries.md for detailed strategy (TBD)

---

## Core Principles (from L2_Strategy_Principles)

All strategy decisions follow these core principles:

1. **Agent-Centric Thinking** - Tracks belong to agents (people), not cameras
2. **Observable Evidence Only** - No statistical priors, no routine modeling
3. **Learn from Data, Then Refine** - Grid learns automatically, config adds semantics
4. **Count Evidence, Don't Do Arithmetic** - No probability scoring
5. **Human-in-Loop Initially** - Trust is earned, automation via learning
6. **Acceptable Errors** - Not all mistakes matter equally
7. **Learn from Clean Data** - Only high-quality observations train the system

**See:** [L2_Strategy_Principles.md](L2_Strategy_Principles.md) for full details

---

## System Lifecycle

The system's lifecycle is defined by:
1. **Bootstrapping** (see L5_Bootstrap.md) - Initial setup, first data collection
2. **Phased Deployment** (see L6_Deployment.md) - Progressive feature rollout

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - Technical architecture (7 components)
- **L2_Strategy_Principles.md** - Core principles guiding all decisions

### Mission & Complexity
- **L1 (Mission):** Why we're solving this problem
- **L4 (Complexity):** Where this strategy gets complicated

### Tactics & Implementation
- **L3_ChunkProcessing.md** - How ChunkProcessor creates tracks (filtering, motion gate, ≥3s threshold)
- **L3_TemporalLinking.md** - How TemporalLinker connects tracks (overlap zone matching, appearance similarity)
- **L3_IdentityResolution.md** - How IdentityResolver merges across cameras (grid learning, alibi checks, process of elimination, face recognition)
- **L3_EvidenceProcessing.md** - How EvidenceProcessor resolves uncertainty (process of elimination, face library, human/LLM review)
- **L3_VehicleOccupancy.md** - Vehicle association tactics (NEEDS MORE SPEC)
- **L3_StructureEntry.md** - Structure entry/exit tactics (NEEDS MORE SPEC)
- **L3_PropertyBoundaries.md** - Property boundary tactics (NEEDS MORE SPEC)
- **L3_Gui.md** - Timeline viewer, review queue
- **L3_Configuration.md** - Portal configuration, thresholds
- **L3_SystemHealth.md** - Performance monitoring

### Technical Specs
- **L10 (Nomenclature):** Terminology definitions
- **L12_Merging:** Cross-camera merge algorithm
- **L12_ProcessOfElimination:** Elimination logic details
- **L12_ConcurrentTracking:** Multi-agent scenarios on single camera
- **L13_Detection:** LocalTrack data structure and creation
