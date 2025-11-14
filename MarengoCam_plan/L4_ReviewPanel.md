# Level 4 - Review Panel (Identity Comparison)

## Purpose
Detailed specification for the identity comparison panel shown to humans/LLMs when reviewing uncertain tracks. This panel helps reviewers decide if two tracks belong to the same person.

**Referenced from:** [L3_EvidenceProcessing.md](L3_EvidenceProcessing.md)

**Implemented in:** L14_ReviewUI.py

---

## Panel Structure

### Overall Layout

```
┌─────────────────────────────────────────────────────────────┐
│ REVIEW PANEL: Track #12345                                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  UNCERTAIN TRACK              CANDIDATE A        CANDIDATE B │
│  ┌──────────────┐            ┌──────────────┐  ┌──────────┐│
│  │ Camera: Driveway          │ Agent: 123    │  │ Agent: 456││
│  │ Time: 10:34:45            │ Name: Agent A │  │ Name: -   ││
│  │ Duration: 4.2s            │ Last seen:    │  │ Last seen:││
│  │                           │ Camera_B      │  │ Camera_C  ││
│  │ [6 best frames]           │ 10:34:21      │  │ 10:33:45  ││
│  │   ...                     │               │  │           ││
│  │                           │ [6 frames]    │  │ [6 frames]││
│  └──────────────┘            └──────────────┘  └──────────┘│
│                                                               │
│  EVIDENCE SUMMARY:                                            │
│  • Travel time: Δt=4.2s (Camera_B → Driveway)                │
│  • Grid timing: Expected 3.8-5.2s (plausible)                │
│  • Face match: Agent A: 0.68 (below threshold)               │
│  •              Agent B: 0.42 (weak)                          │
│  • Alibi check: 2 alternative sources (ambiguous)            │
│  • Re-ID similarity: Agent A: 0.73, Agent B: 0.51            │
│                                                               │
│  DECISION:                                                    │
│  [ Merge with Agent A ]  [ Merge with Agent B ]              │
│  [ Reject both ]         [ Create new agent ]  [ Skip ]      │
│                                                               │
│  REASONING (optional):                                        │
│  [Text box for explanation]                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Frame Selection (6 Best Frames)

### Selection Criteria

For each track (uncertain + candidates), show **6 best frames**:

1. **Face visibility priority:**
   - Front-facing > Side profile > Occluded
   - Face detection confidence threshold: ≥ 0.6

2. **Body visibility:**
   - Full body visible > Partial > Cropped

3. **Image quality:**
   - Sharp focus > Motion blur
   - Good lighting > Poor lighting

4. **Temporal distribution:**
   - Spread across track duration (beginning, middle, end)
   - Avoid consecutive frames (show distinct poses)

5. **Clothing/appearance:**
   - Prioritize frames showing distinctive clothing features
   - Include frames with accessories (hats, bags, etc.)

### Frame Layout

**6 frames arranged in 2 rows × 3 columns:**
```
┌─────┬─────┬─────┐
│  1  │  2  │  3  │  (beginning → middle)
├─────┼─────┼─────┤
│  4  │  5  │  6  │  (middle → end)
└─────┴─────┴─────┘
```

**Each frame includes:**
- Timestamp overlay (bottom-right corner)
- Bounding box highlight (optional, configurable)
- Frame quality indicator (confidence score)

---

## Evidence Summary

### Required Fields

**Spatial/Temporal Context:**
- **Travel time:** `Δt = track_B.start_time - track_A.end_time`
  - Format: "4.2s"
  - Color-coded: Green (< 5s), Yellow (5-30s), Red (> 30s)

- **Grid timing:** Expected range based on learned grid
  - Format: "Expected 3.8-5.2s (plausible)" or "Expected 2.1-3.5s (too slow)"
  - Status: "Plausible", "Too fast", "Too slow", "Overlap zone"

**Identity Evidence:**
- **Face match confidence:** Per candidate, from CodeProject.AI
  - Format: "Agent A: 0.68 (below threshold)"
  - Threshold reference: ≥ 0.75 for auto-merge

- **Re-ID similarity:** Body appearance embedding similarity
  - Format: "Agent A: 0.73, Agent B: 0.51"
  - Scale: 0.0-1.0 (higher = more similar)

**Disambiguation Context:**
- **Alibi check:** Number of alternative sources
  - Format: "2 alternative sources (ambiguous)" or "No alternatives (definitive)"
  - Indicates whether other tracks could have reached this location

- **Concurrent activity:** Number of people visible simultaneously
  - Format: "3 concurrent agents (high ambiguity)"

### Optional Fields (if available)

- **Previous/next track context:** "Previously seen in Camera_A at T=10:30:12"
- **Portal transitions:** "Crossed garage_door portal (time gap irrelevant)"
- **Clothing similarity:** "Red shirt + blue jeans (matches Agent A)"

---

## Decision Actions

### Primary Actions

**Merge with Candidate:**
- Assigns `track.global_agent_id = candidate.agent_id`
- Clears `track.identity_candidates`
- Adds to face library (validated merge)
- Propagates through relationships
- Triggers process of elimination

**Reject Candidate:**
- Removes candidate from `track.identity_candidates`
- Does NOT assign identity
- If only one candidate remains → auto-resolve
- Provides negative training example

**Create New Agent:**
- Generates new `global_agent_id`
- Assigns to track
- Creates new face library entry
- Use when track clearly doesn't match any candidate

### Secondary Actions

**Skip/Defer:**
- Leaves track in review queue
- No changes to identity_candidates
- Used when reviewer is uncertain or needs more context

**Request More Context:**
- Shows additional frames (expand from 6 to 12)
- Shows prev/next tracks in timeline
- Shows concurrent tracks at same timestamp

---

## Reasoning Field

### Purpose
Captures human/LLM explanation for decision, useful for:
- Training data annotation
- Debugging auto-merge failures
- Understanding edge cases
- System improvement

### Format
Free-form text, optional but encouraged

**Example reasoning:**
- "Same red jacket and gait pattern, confident match"
- "Face partially occluded but clothing and timing match Agent A"
- "Too many people in garage, cannot determine identity"
- "Agent B was seen elsewhere (alibi check failed)"

---

## Panel Variations

### For LLM Review
**Differences from human panel:**
- Structured prompt format (not visual UI)
- Pre-computed similarity scores emphasized
- Reasoning field required (not optional)
- Output format: JSON decision

**Example LLM prompt:**
```
You are reviewing track #12345 (Camera: Driveway, T=10:34:45, Duration=4.2s)

CANDIDATES:
- Agent A (ID: 123): Last seen Camera_B at 10:34:21 (Δt=4.2s)
- Agent B (ID: 456): Last seen Camera_C at 10:33:45 (Δt=8.0s)

EVIDENCE:
- Grid timing: Expected 3.8-5.2s for Camera_B→Driveway (Agent A plausible)
- Face match: Agent A: 0.68 (below threshold), Agent B: 0.42 (weak)
- Re-ID similarity: Agent A: 0.73, Agent B: 0.51
- Alibi check: 2 alternative sources (ambiguous)

IMAGES: [6 frames uncertain track] | [6 frames Agent A] | [6 frames Agent B]

DECISION: Merge with Agent A / Merge with Agent B / Reject both / Uncertain
CONFIDENCE: high / medium / low
REASONING: [Your explanation]
```

### For High-Priority Reviews
**Enhancements:**
- Show more context (prev/next tracks)
- Show concurrent tracks
- Show map visualization (grid path)
- Expand to 12 frames per track

---

## Related Documents

### Tactical Context
- **L3_EvidenceProcessing.md** - Review queue management, panel generation logic
- **L3_IdentityResolution.md** - Evidence hierarchy, decision thresholds
- **L3_Gui.md** - Overall GUI structure

### Implementation
- **L14_ReviewUI.py** - Panel rendering, frame selection, user interaction
- **L14_PanelAssemblyService.py** - Frame selection algorithm, evidence computation

### Algorithms
- **L12_ProcessOfElimination.md** - Alibi check computation
- **L12_GridLearning.md** - Grid timing computation
