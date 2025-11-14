# Documentation Audit: Vague, Underspecified, and Flawed Areas

**Date:** 2025-11-14
**Scope:** L1-L6, L10-L13 planning documents
**Purpose:** Identify gaps, inconsistencies, and design flaws requiring clarification

---

## CRITICAL ISSUES

### 1. Entry/Exit Cell Definition (L10 TBD)
**Severity:** HIGH
**Location:** L10_Nomenclature.md:134-138

**Issue:**
```
TBD: The precise logic for determining the entry cell (e.g., first detection, or first stable detection) needs to be defined.
TBD: The precise logic for determining the exit cell (e.g., last detection) needs to be defined.
```

**Why it matters:**
- Grid transition logic depends on entry/exit cells
- Different definitions → different learned times
- Example: If exit cell = "last frame with ≥0.5 confidence" vs "last stable detection", grid times differ
- Affects auto-merge decisions (grid overlap detection)

**Current guess from code:**
- L13_Detection.md doesn't specify
- L12_Merging.md uses `track.end_cell` and `track.start_cell` but doesn't define them

**What's needed:**
```python
# Option A: Simple boundary-based
entry_cell = grid_cell(detections[0].center)
exit_cell = grid_cell(detections[-1].center)

# Option B: Stable-based (threshold-dependent)
stable_threshold = 3  # frames
entry_cell = grid_cell(detections[stable_threshold].center)
exit_cell = grid_cell(detections[-stable_threshold].center)

# Option C: Quality-based
entry_cell = grid_cell(detections[first_confident].center)  # conf > 0.7
exit_cell = grid_cell(detections[last_confident].center)
```

**Impact:** Will affect all cross-camera merging accuracy

---

### 2. "Process of Elimination" Undefined (L2/L3)
**Severity:** HIGH
**Location:** L2_Strategy.md:140-142, L3_TrackMerging.md:13

**Issue:**
```
Process of elimination: Agent A seen elsewhere → rules out Track X belonging to Agent A
Cascades: ruling out candidates increases confidence in remaining
```

**Why it matters:**
- Sounds simple conceptually, but implementation is complex
- Example: Agent A at camera A at 12:03, track X at camera B at 12:05
  - How far apart is "elsewhere"?
  - What's the time window? (A to B takes 30s, so A can't be X if A is still at B at 12:05)
  - Does this work across 10 cameras? What's the search radius?

**Current state:** Never specified. Assumed to be in L13 but not found.

**What's needed:**
```python
def process_of_elimination(candidate_track, agent_id, system_agents):
    """
    Check if agent_id could possibly be the agent in candidate_track.

    Rules:
    1. Agent's last known location vs candidate start location
    2. Travel time plausibility (can agent reach candidate in time?)
    3. Does agent have alibi? (seen elsewhere simultaneously?)
    4. Portal crossing timing (exited portal, entering different portal)?
    """
    # Not specified: time window, distance threshold, etc.
    pass
```

**Impact:** Multi-candidate disambiguation, vehicle occupancy resolution

---

### 3. Grid Overlap Threshold Undefined (L12 code vs L2 strategy)
**Severity:** HIGH
**Location:** L12_Merging.md:43, L2_Strategy.md:33

**Issue:**
- L2 says: "overlap zones (typical_time ≈ 0)" enable "high-confidence auto-merges"
- L12 code: `if grid_stats.is_overlap(): # typical_time < 1.0`
- **Question:** Is "overlap" < 1.0 seconds? < 500ms? < 100ms?

**Why it matters:**
- Too loose: False auto-merges (person A exits frame, person B enters from static area → time = 0 but different people)
- Too strict: Legitimate overlaps rejected (person walks corner, appears static for 1.2s)
- No learning baseline specified: what counts as first overlap data point?

**Current state:** L10 says `0` indicates overlap, L12 uses `< 1.0` without explanation

**What's needed:**
```python
# In L12 or L11:
GRID_OVERLAP_THRESHOLD = 0.5  # seconds
GRID_OVERLAP_MIN_SAMPLES = 5   # before auto-merge allowed

def is_overlap(grid_stats):
    """Overlap if typical time < 500ms AND observed at least 5 times."""
    return grid_stats.typical_time < GRID_OVERLAP_THRESHOLD and \
           grid_stats.sample_count >= GRID_OVERLAP_MIN_SAMPLES
```

**Impact:** Auto-merge false positive rate, grid learning correctness

---

### 4. Multi-Assignment Merging Semantics Unclear (L2/L13)
**Severity:** HIGH
**Location:** L2_Strategy.md:135-138, L13_TrackMerging.md (not found)

**Issue:**
```
Track marked as "possibly Agent A or Agent B" (no probabilities)
Timeline shows all possibilities
User can manually resolve if needed
```

**What's actually underspecified:**
- How are multi-assignments stored? (One agent_id? Multiple? Probability weights?)
- When does a multi-assignment become definitive? (User choice? Later evidence?)
- How does multi-assignment affect downstream decisions? (Timeline visibility? Merge candidates? Re-ID learning?)
- Can a multi-assignment persist indefinitely, or must it eventually resolve?

**Database representation TBD:**
- L13_Database.md mentions `candidate_agents TEXT` but no algorithm for propagating/resolving

**What's needed:**
```python
class MultiAssignment:
    """A track that might belong to multiple agents."""
    track_id: str
    candidate_agents: List[str]  # [A, B, C]
    evidence_per_agent: Dict[str, List[str]]  # A: [face_match, grid], B: [timing]

    def is_resolvable(self) -> bool:
        """Check if later evidence can disambiguate."""
        pass

    def resolve_to(self, agent_id: str):
        """User/system picks one agent."""
        pass
```

**Impact:** Timeline ambiguity handling, user UX for review queue

---

### 5. Re-ID Model Choice Missing (L12)
**Severity:** MEDIUM
**Location:** L12_Merging.md:351

**Issue:**
```
Note: Re-ID model TBD (osnet_x1_0, resnet50_ibn, etc.)
# TODO: Implement Re-ID model inference
```

**Why it matters:**
- L2 strategy mentions Re-ID as secondary evidence
- L12 code uses `reid_similarity` score but doesn't specify:
  - Which model?
  - How to compute similarity? (cosine distance? L2? threshold?)
  - Where does it run? (Inference Manager? CodeProject.AI? Local?)
  - What's the threshold for "high-confidence" vs "ambiguous"?

**Current state:** Completely unspecified, blocking L12 implementation

**What's needed:**
```python
# L11 or L12:
RE_ID_MODEL = "osnet_x1_0"  # or resnet50_ibn
RE_ID_THRESHOLD_AUTO = 0.85
RE_ID_THRESHOLD_REVIEW = 0.65

def compute_reid_similarity(track1: Track, track2: Track) -> float:
    """Cosine distance between Re-ID embeddings."""
    # Not specified: which crops to use? Best quality? Multiple?
    pass
```

**Impact:** Vehicle merging, person merging when face occluded

---

### 6. ARM Timeout Value Undefined (L3/L10)
**Severity:** MEDIUM
**Location:** L10_Nomenclature.md:227, L3_DetectionAndTracking.md

**Issue:**
```
Duration: post_roll_s (10-15 seconds, TBD)
```

Also added in L4_Complexity.md:
```
Reasonable ARM timeout (e.g., 15-30 seconds)
```

**What's missing:**
- Is it 10s? 15s? 30s? Different per class (person vs vehicle)?
- How does this interact with motion gate re-arming?
  - If motion stops at 8s and person pauses, does ARM reset?
  - Or does ARM expire regardless?

**Impact:** False activity splits (person pauses → two agents), resource waste

---

### 7. Motion Gate Pixel Threshold Undefined (L2/L13)
**Severity:** MEDIUM
**Location:** L2_Strategy.md:75-78

**Issue:**
```
Motion gate (Standby only):
- Fast pixel-diff check before YOLO inference
- Filters static frames (no motion = no inference)
- Only frames with motion delta above threshold proceed to sparse YOLO sampling
```

**What's missing:**
- Pixel difference threshold? (e.g., 5% of frame? 10%?)
- Sensitivity per lighting? (Night mode vs day?)
- Kernel size for motion detection? (3x3? 5x5? Temporal?)
- How does this threshold relate to noise (wind, shadows)?

**Impact:** False negatives (real motion filtered), false positives (shadows trigger YOLO)

---

### 8. Movement Score Thresholds Undefined (L13/L4)
**Severity:** MEDIUM
**Location:** L13_TrackStateManager.md:31, L4_Complexity.md:200

**Issue:**
```python
self.movement_score > MOVEMENT_THRESHOLD  # MOVEMENT_THRESHOLD not defined
```

```
Start conservative (3.0s + 0.6 movement score)
```

**What's missing:**
- Is threshold 0.6? 0.5? Per-class (person 0.6, vehicle 0.4)?
- MIN_FRAME_COUNT? (I used `MIN_FRAME_COUNT` in code but no value)
- How is movement_score calculated from my function? (Normalize differently?)

**Impact:** Candidate promotion rate, noise filtering effectiveness

---

### 9. Portal Timing Variance Margin Unclear (L12)
**Severity:** MEDIUM
**Location:** L12_Merging.md:68

**Issue:**
```python
margin = max(2.0, grid_stats.variance * 2)  # At least 2s margin
```

**What's missing:**
- Why multiply variance by 2? Statistical justification?
- Why minimum of 2.0s? What if variance is 0.5s?
- Does this work for all portals? (Fast hallway vs slow stairs?)
- How does this affect learning? (Do outliers corrupt variance calculation?)

**Impact:** Portal transition acceptance/rejection, merge candidate quality

---

### 10. Face Similarity Threshold for Registration (L2/L12)
**Severity:** MEDIUM
**Location:** L2_Strategy.md:68, L12_Merging.md:34

**Issue:**
```
Face library builds from high-confidence merges only (face recognition ≥ threshold)
if face_match(track1, track2) > 0.75:
```

**What's missing:**
- Is threshold 0.75 or just example?
- Different for registration vs matching? (Register at 0.85, match at 0.70?)
- Per-quality? (High-quality faces need higher threshold?)
- How does CodeProject.AI confidence map to this? (Different scoring system?)

**Impact:** Face library quality, cascading errors from wrong registrations

---

### 11. LLM Decision Format Undefined (L2/L3)
**Severity:** MEDIUM
**Location:** L2_Strategy.md:35, L3_TrackMerging.md:17-18

**Issue:**
```
Check for Conflicting Evidence: Does the evidence conflict? If yes, REVIEW_HUMAN.
```

**What's missing:**
- What prompts LLM? What's in the context?
- How are images formatted for LLM? (Resolution? Crops? How many?)
- LLM response format? (Yes/No? Confidence scores? Reasoning?)
- What counts as LLM "cannot"? (L10 mentions it but no triggering conditions)
- Fallback if LLM fails? (Timeout? Error? Ask human anyway?)

**Impact:** LLM review queue, cost/latency, decision quality

---

### 12. "Plausible" Grid Transition Definition (L2/L3/L12)
**Severity:** MEDIUM
**Location:** L2_Strategy.md:34, L3_TrackMerging.md:11, L12_Merging.md:16

**Issue:**
```
High-Confidence Time/Space Match (validated grid overlap with single candidate)
Plausible Time/Space Match (learned grid adjacency, evaluated with other evidence)
```

**What's missing:**
- Exact difference between "High-Confidence" and "Plausible"?
- Is "plausible" = within learned time ± 2× variance?
- What if grid has no data? (Unlearned path = plausible or rejected?)
- How many samples needed before grid is "learned"? (5? 10? 50?)

**Impact:** Auto-merge vs review queue assignment

---

### 13. Candidate Track Buffer Size (L13/L3)
**Severity:** LOW
**Location:** L13_TrackStateManager.md (implied), not explicitly stated

**Issue:**
```
detections: List[Detection]  # How many?
```

**What's missing:**
- Keep all detections in memory? (25s buffer = 250 frames per camera)
- Store only summary stats (duration, movement_score, count)?
- Actual JPEG bytes or just Detection metadata?
- Memory impact: 10 cameras × 250 frames × 10KB/frame = 25MB per camera = 250MB total

**Impact:** Memory footprint, streaming vs batch processing choice

---

### 14. Portal Entry/Exit Definition (L3)
**Severity:** LOW
**Location:** L3_EvidenceEngine.md:17, L3_TrackMerging.md:27-31

**Issue:**
```
Does a configured portal connect the exit cell of the source track and the entry cell of the destination track?
When a track disappears into a Portal and a new track later emerges from the same Portal, transition is plausible
```

**What's missing:**
- Is portal = single cell? Or region?
- How close does exit cell need to be to portal? (Exact match? Adjacent?)
- If person exits at cell (3,4) and portal is at (3,3), is it "same portal"?

**Impact:** Portal matching logic, false positives in merging

---

### 15. Grid Learning "Clean Data" Criteria (L2/L4)
**Severity:** MEDIUM
**Location:** L2_Strategy_Principles.md:59-63, L4_Complexity.md:254-269

**Issue:**
```
Only update core system knowledge from high-confidence, unambiguous, single-agent observations
Merges involving groups or uncertainty excluded from learning
```

**What's missing:**
- Exact criteria for "high-confidence"? (Face match? LLM validated? Human approved?)
- How to detect "single-agent"? (Count agents in overlap? Timeline review?)
- What if person paused mid-transition? (Valid for learning or contaminated?)
- How to mark a merge as "exclude from learning"?

**Current state:** L13_LearningAndTraining.md likely has this, but not yet reviewed

**Impact:** Grid data quality degradation over time, merging accuracy drift

---

## DESIGN FLAWS / INCONSISTENCIES

### 16. Static Agent Detection Contradiction (L2)
**Severity:** MEDIUM
**Location:** L2_Strategy.md:82-84

**Quote:**
```
Static objects DO NOT generate active tracks - stationary agents exist but can't be merged
Example: Parked car is an agent with state visible_parked, but has no active track
```

**Issue:**
- Agents can exist without tracks (`visible_parked` = agent exists)
- But agents need tracks to be merged (and thus appear in timeline)
- **Question:** How does `visible_parked` agent get into timeline if no track?
- **Question:** Can user see "parked car" event if no track? (Seems impossible with current architecture)
- **Question:** When car starts moving, is it a new agent or same agent? (Timeline continuity broken if new)

**Impact:** Parked vehicle tracking, timeline reconstruction for static agents

---

### 17. In-Vehicle Person Exemption Confusing (L2/L12)
**Severity:** MEDIUM
**Location:** L2_Strategy.md:189-194, L12_Merging.md:48-50

**Issue:**
```
Person with in_vehicle(V) or visible_in_vehicle(V) state exempt from normal cross-camera merge rules
Can't walk camera-to-camera while in car (vehicle does the moving)
```

**But how does system detect entry?**
```
Person track ends near vehicle → mark as in_vehicle
```

**Contradiction:**
- Entry: "Person track ends + vehicle moves" → in_vehicle (temporal/spatial logic)
- Exit: "Vehicle parks + person appears" → merge candidate
- **What if:** Person enters vehicle at edge of frame, only 0.5s track before occlusion?
  - Does candidate track get created? Promoted?
  - Or does "in_vehicle" state prevent it?

**Impact:** Vehicle occupancy tracking accuracy, person identity continuity

---

### 18. Face Library Update Criteria Loosened (L2 vs L4)
**Severity:** MEDIUM
**Location:** L2_Strategy.md:68-70 vs L4_Complexity.md:254-269

**L2 says (new):**
```
Face library builds from high-confidence merges only (face recognition ≥ threshold OR validated human review)
Uncertain merges refine timeline tracking but don't pollute face library
```

**L4 (existing risk) says:**
```
Conservative auto-merge thresholds initially
Human validation before face registration
```

**Inconsistency:**
- L2 = "high-confidence OR human validated"
- L4 = "human validation" (implies face registration only after human approval)
- Which is it? If someone human-validates a merge, face auto-registers? Or require explicit "register this face" action?

**Impact:** Face library quality over time

---

### 19. Candidate Track vs Motion Gate Interaction Unclear (L3)
**Severity:** MEDIUM
**Location:** L3_DetectionAndTracking.md:21-25, L13_TrackStateManager.md:42

**Issue:**
```
Motion gate (Standby only):
- Only frames with motion delta above threshold proceed to sparse YOLO sampling
```

```
Accumulation (Armed): Each frame: accumulate detection, update duration, compute movement_score
```

**Question:**
- During Armed, does motion gate still run? (If motion stops, does candidate discard?)
- Or does Armed disable motion gate? (Any detection continues accumulation?)
- If person pauses mid-walk, is that a detection? Does ARM timeout still tick?

**Impact:** Candidate lifetime, pause handling (person briefly stops)

---

## MISSING SPECIFICATIONS

### 20. Portal Configuration Format (L3/L13)
**Severity:** LOW
**Location:** Not found (assumed in L3_Configuration.md or L13_Configuration.md)

**What's needed:**
```yaml
portals:
  - name: "front_door"
    camera_pair: ["FrontDoor", "Hallway"]
    entry_cells: [[3, 4], [3, 5]]  # Cameras entry regions
    exit_cells: [[2, 4], [2, 5]]   # Cameras exit regions
    max_time_seconds: 5

  - name: "garage"
    type: "terminal"  # Exit portal
    camera: "FrontDoor"
    exit_cells: [[5, 5], [5, 6]]
    description: "Garage door"
```

---

### 21. Inference Manager Priority/Budget Allocation (L3/L13)
**Severity:** MEDIUM
**Location:** L3_InferenceManager.md (only stub), L13_InferenceManager.md (not reviewed)

**What's missing:**
- How does Inference Manager allocate GPU budget across cameras?
- Priority formula: Active > Armed > Post > Standby?
- Can Active camera "starve" Standby? (How is fairness ensured?)
- What happens during Suppressed state? (How much reduction?)

---

### 22. Timeline Reconstruction Algorithm (L3/L13)
**Severity:** MEDIUM
**Location:** L3_TimelineReconstruction.md (not reviewed), L13_TimelineReconstruction.md (not reviewed)

**What's missing:**
- How are fragmented tracks stitched into agent journey?
- Multi-assignment resolution: do all possibilities show, or pick most likely?
- Occlusion handling: when agent is `unknown`, how is gap bridged?
- Portal crossing inference: when person exits via portal, infer `in_structure`?

---

## SUMMARY TABLE

| Issue | Severity | Category | File | Fix Priority |
|-------|----------|----------|------|--------------|
| Entry/Exit Cell Logic | HIGH | Specification | L10 | 1 (Merging depends) |
| Process of Elimination | HIGH | Algorithm | L2/L3/L13 | 2 (Multi-assignment) |
| Grid Overlap Threshold | HIGH | Tuning | L12/L10 | 3 (Auto-merge accuracy) |
| Multi-Assignment Semantics | HIGH | Specification | L2/L13 | 4 (Timeline/merging) |
| Re-ID Model Choice | MEDIUM | Implementation | L12 | 5 (Vehicle merge) |
| ARM Timeout | MEDIUM | Tuning | L10/L3 | 6 (Candidate lifecycle) |
| Motion Gate Threshold | MEDIUM | Tuning | L2/L13 | 7 (Noise filtering) |
| Movement Score Threshold | MEDIUM | Tuning | L13/L4 | 8 (Promotion criteria) |
| Portal Timing Variance | MEDIUM | Algorithm | L12 | 9 (Grid matching) |
| Face Threshold | MEDIUM | Tuning | L2/L12 | 10 (Face library) |
| LLM Interface | MEDIUM | Specification | L2/L3 | 11 (Review queue) |
| "Plausible" Definition | MEDIUM | Specification | L2/L3/L12 | 12 (Merging rules) |
| Candidate Buffer | LOW | Architecture | L13 | 13 (Memory) |
| Portal Geometry | LOW | Configuration | L3 | 14 (Portal matching) |
| Learning Criteria | MEDIUM | Specification | L2/L4/L13 | 15 (Grid quality) |
| Static Agent Timeline | MEDIUM | Design | L2 | 16 (Use case) |
| In-Vehicle Exemption | MEDIUM | Specification | L2/L12 | 17 (Vehicle tracking) |
| Face Registration Criteria | MEDIUM | Specification | L2/L4 | 18 (Face library) |
| Motion Gate/ARM Interaction | MEDIUM | Specification | L3 | 19 (Candidate lifecycle) |
| Portal Config Format | LOW | Specification | L13 | 20 (Admin) |
| Inference Allocation | MEDIUM | Algorithm | L3/L13 | 21 (GPU management) |
| Timeline Reconstruction | MEDIUM | Algorithm | L3/L13 | 22 (Output) |

---

## RECOMMENDATIONS

1. **Create L13 subsection for all TUNING values:** Consolidate all thresholds (motion gate, movement score, ARM timeout, face thresholds, portal variance margin, grid overlap threshold, Re-ID thresholds)

2. **Create L13 algorithm sections:** Process of elimination, portal matching logic, face library registration decision, timeline reconstruction stitching

3. **Clarify L2 contradiction on static agents:** Can they appear in timeline without tracks? If yes, how?

4. **Define Re-ID model + inference location:** Pick osnet or resnet50, specify embedding dimension, similarity metric, thresholds

5. **Multi-assignment data model:** Design exact database schema and resolution algorithm
