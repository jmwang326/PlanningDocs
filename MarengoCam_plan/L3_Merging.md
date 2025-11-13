# Level 3 - Cross-Camera Merging

## Purpose
How we merge track segments across cameras to build agent timelines.

## Merge Candidates

### Finding Candidates
**Periodic matching** (e.g., every 30 seconds):
- Compare recent track segments across all cameras
- Time window: tracks within ~60s of each other
- Portal configuration narrows candidates (user-defined connections)

### Portal Configuration
**User manually defines camera connections:**
- Which cameras connect (door, gate, driveway)
- Expected transition locations (cells/zones on each camera)
- Typical timing (optional, system learns this)

**System uses portals to:**
- Filter candidates (only check connected cameras)
- Learn transition timing from validated merges
- Reject impossible transitions (too fast/slow)

## Evidence Evaluation

### Evidence Types

**Strong Evidence:**
- Face match (known person from library)
- Portal crossing (track ends at portal, next starts at corresponding portal)
- Only candidate (no other plausible options)
- Time/space exact fit

**Weak Evidence:**
- Spatial proximity (nearby locations)
- Timing fits (plausible travel time)
- Visual similarity (clothing, size)

### Decision Hierarchy

**1. Face Recognition** (fastest, most definitive)
- Match against face library
- Known person → auto-merge
- Unknown/no face → continue to next step

**2. Time/Space Gating** (eliminate impossible)
- Too fast: teleportation (reject)
- Too slow: implausible (reject)
- Candidates that pass: continue

**3. Portal Check** (user configuration)
- Track ends at portal on Camera A
- Next track starts at corresponding portal on Camera B
- Timing within learned window → high confidence candidate

**4. LLM Adjudication** (expensive, ambiguous cases)
- Present image panels from both tracks
- Ask: "Same person/vehicle?"
- Forced-choice: YES/NO/CANNOT

**5. Human Validation** (final fallback)
- Queue for manual review
- User confirms or rejects merge
- System learns from decision

## Handling Uncertainty

### Multi-Assignment
**Track may belong to multiple agents:**
- Two people walk through portal together
- Tracks split on other side
- System marks: "Track X possibly Agent A (60%) or Agent B (40%)"

### Uncertainty Persistence
**Don't force wrong merges:**
- Uncertain assignment can persist indefinitely
- Timeline shows possibilities
- Later evidence may resolve (face visible, process of elimination)

### Process of Elimination
**Ruling out candidates increases confidence:**
- Agent A seen elsewhere at same time → rules out Track X = Agent A
- Only one candidate remains → confidence increases
- Cascading updates as new evidence appears

## Learning from Validations

### Face Library
**Builds over time:**
- Human/LLM validates merge → register face
- Next appearance: auto-merge via face match
- Known people require less LLM/human review

### Portal Timing
**Tightens windows:**
- Initial: wide window (max_time = 10s)
- After validations: learn typical_time (e.g., 2s ± 1s)
- Reject candidates outside learned window

## Related Documents
- **L2 (Strategy):** Why evidence hierarchy and uncertainty handling
- **L4 (Complexity):** Where merging gets complicated
- **L11_Merging:** Technical specs (algorithms, data structures)
