# Level 2 - Strategy

## Purpose
Non-technical description of **what** we're doing and **why** this approach solves the problem.

## Strategy Overview

### Filter Noise First
**What:** Only promote sustained movement to "real activity"
**Why:** Trees/shadows/brief glitches create false alerts
**Approach:** Object must move continuously (not just detected once)

### Track Within Cameras
**What:** Follow objects frame-to-frame within single camera
**Why:** YOLO11 provides tracking, reduces per-frame detections to persistent tracks
**Approach:** Tracker links detections across frames, creates track segments

### Merge Across Cameras
**What:** Link track segments that belong to same agent
**Why:** Timeline requires knowing when Track A (Camera 1) = Track B (Camera 2)
**Approach:** Evidence-based matching (face, time/space, portals)

### Evidence Hierarchy
**What:** Use strongest evidence first, fallback to weaker evidence
**Why:** Face match is definitive, time/space is suggestive, LLM is expensive

**Hierarchy:**
1. **Face recognition** (known people auto-merge)
2. **Time/space gating** (impossible transitions rejected)
3. **LLM adjudication** (ambiguous cases)
4. **Human validation** (final fallback)

### Accept Uncertainty
**What:** Some tracks can't be assigned definitively
**Why:** Evidence may be missing (face occluded, timing ambiguous, multiple candidates)

**Approach:**
- Mark tracks as "possibly Agent A or Agent B"
- Uncertainty persists until resolved by later evidence
- System operates with partial timeline
- Don't force wrong merge to avoid uncertainty

### Learn from Corrections
**What:** Human/LLM validations improve future decisions
**Why:** System learns what "same person" looks like over time

**Approach:**
- Face library builds from validated merges
- Face recognition improves as library grows

## Strategy for Noise Filtering

### Real vs Noise
**What determines real activity:**
- Object type: person, vehicle, animal (not "unknown")
- Persistence: tracked across multiple frames
- Movement: sustained motion (not static detection)

**What gets filtered:**
- Brief detections (< threshold duration)
- Low confidence objects
- Stationary objects (trees, parked cars)
- Detector glitches

### Timeline Construction
**Goal:** "What happened today?" answered by agent timeline, not camera-by-camera logs

**Approach:**
- Merge tracks into agent journeys
- Show agent movement across property
- Filter out noise (never promoted to timeline)

## Strategy for Uncertainty

### Types of Uncertainty

**Identity Uncertainty:**
- Track might belong to Agent A or Agent B
- Example: Two people walk through portal together, tracks split on other side

**Location Uncertainty:**
- Agent exists but current location unknown
- Example: Person enters building, not visible for 10 minutes

**Assignment Uncertainty:**
- Can't determine if Track X belongs to existing agent or is new agent
- Example: Face occluded, timing fits multiple candidates

### Handling Uncertainty

**Don't force decisions:**
- Wrong merge worse than uncertain merge
- Uncertainty can persist indefinitely
- Later evidence may resolve (person exits building, face visible)

**Multi-assignment possible:**
- Track marked as "possibly Agent A or Agent B" (no probabilities)
- Timeline shows all possibilities
- User can manually resolve if needed

**Process of elimination:**
- Agent A seen elsewhere → rules out Track X belonging to Agent A
- Cascades: ruling out candidates increases confidence in remaining

## Related Documents
- **L1 (Mission):** Why we're solving this problem (core values, goals)
- **L3 (Tactics):** How components implement this strategy
- **L4 (Complexity):** Where this strategy gets complicated
- **L10 (Nomenclature):** Terminology definitions
