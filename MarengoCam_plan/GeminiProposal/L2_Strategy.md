# Level 2 - Strategy

## Purpose
Non-technical description of **what** we're doing and **why** this approach solves the problem.

**See also:** [L2_Architecture.md](L2_Architecture.md) for the technical architecture (Hub-and-Spoke model).

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

**See:** [L3_ChunkProcessing.md](L3_ChunkProcessing.md) for filtering tactics

---

### 2. Link Tracks Over Time
**Why:** YOLO assigns new IDs each chunk. We need continuous observation on one camera.

**Approach:**
- 2s overlap between chunks
- Match by position + appearance in overlap zone
- Build linked list of tracks (previous/next pointers)

**Result:** Continuous timeline on one camera, even though YOLO IDs change.

**See:** [L3_TemporalLinking.md](L3_TemporalLinking.md) for linking tactics

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

**See:** [L3_IdentityResolution.md](L3_IdentityResolution.md) for merging tactics (grid learning, alibi checks, process of elimination)

---

### 4. Accept Uncertainty
**Why:** Evidence may be missing (face occluded, timing ambiguous, multiple candidates).

**Approach:**
- Don't force wrong merge to avoid uncertainty
- Mark track with `identity_candidates` set
- Uncertainty persists until evidence resolves it
- System operates with partial timeline

**Result:** Never force wrong decisions. Wait for evidence.

**See:** [L3_EvidenceProcessing.md](L3_EvidenceProcessing.md) for uncertainty resolution tactics

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

**See:** [L3_IdentityResolution.md](L3_IdentityResolution.md) for grid learning tactics

---

### 6. No Probability Scores
**Why:** Can't compute accurate weights (0.3×time + 0.5×face = nonsense). Can't debug ("why 0.65?").

**Approach:**
- `identity_candidates` is a **set** (not probabilities)
- Evidence is countable and explicit
- Process of elimination removes impossible candidates

**Result:** Debuggable, transparent decisions.

**See:** [L3_EvidenceProcessing.md](L3_EvidenceProcessing.md) for process of elimination tactics

---

## Related Documents

### Architecture
- **[L2_Architecture.md](L2_Architecture.md)** - Technical architecture

### Mission
- **[L1_Mission.md](L1_Mission.md)** - Why we're solving this problem

### Tactics
- **[L3_ChunkProcessing.md](L3_ChunkProcessing.md)** - How ChunkProcessor creates tracks
- **[L3_TemporalLinking.md](L3_TemporalLinking.md)** - How TemporalLinker connects tracks
- **[L3_IdentityResolution.md](L3_IdentityResolution.md)** - How IdentityResolver merges across cameras
- **[L3_EvidenceProcessing.md](L3_EvidenceProcessing.md)** - How EvidenceProcessor resolves uncertainty
- **[L3_Gui.md](L3_Gui.md)** - Timeline viewer, review queue
- **[L3_Configuration.md](L3_Configuration.md)** - Portal configuration, thresholds
- **[L3_SystemHealth.md](L3_SystemHealth.md)** - Performance monitoring

### Reference
- **[L1T_Nomenclature.md](L1T_Nomenclature.md)** - Terminology definitions
