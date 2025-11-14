# Level 4 - Complexity and Risks

## Purpose
What's hard, what's tuning-dependent, and where failures occur.

## Merge Complexity

### Multi-Person Scenarios
**HARD: 2+ people moving together**

**Challenge:**
- Two people walk through portal together
- Tracks split on Camera B
- Which Track A person → which Track B person?

**Risk:**
- ID swap (Person 1 merged with wrong track)
- Breaks timeline for both people

**Mitigation:**
- Multi-assignment: Track A₁ possibly (B₁ or B₂), Track A₂ possibly (B₁ or B₂)
- Face match can resolve ambiguity
- Don't use ambiguous merges for grid learning (contaminated data)
- Human/LLM review if needed

---

### Occlusions
**HARD: Agent disappears mid-transition**

**Challenge:**
- Person walks toward portal on Camera A
- Portal area has occlusion (tree, wall)
- Next appearance on Camera B delayed or missed

**Risk:**
- Track ends early on Camera A
- Track starts late on Camera B
- Time window too wide → false candidates

**Mitigation:**
- Wide initial time windows
- Learn from clean observations only
- Accept uncertainty if ambiguous

---

### Similar Appearance
**TUNING: Clothing/vehicle similarity**

**Challenge:**
- Two people wearing dark clothing
- Same make/model vehicles
- Face occluded or poor quality

**Risk:**
- Wrong merge based on visual similarity alone

**Mitigation:**
- Rely on time/space gating first (filter impossible)
- LLM adjudication for remaining ambiguity
- Don't auto-merge on visual similarity alone

---

## Thresholds and Tuning

### Confidence Thresholds
**TUNING: Detector confidence (0.5? 0.6? 0.7?)**

**Too low:** False positives (shadows, noise promoted to tracks)
**Too high:** Misses real activity (low-light, distant objects)

**Approach:** Start conservative, calibrate from real footage

---

### Duration Thresholds
**TUNING: How long = "real activity"? (2s? 3s? 5s?)**

**Too low:** Brief noise promoted to events
**Too high:** Quick legitimate activity missed

**Approach:** Different by class? (person 3s, vehicle 5s, animal 2s)

---

### Portal Timing Windows
**TUNING: Max travel time between cameras**

**Too narrow:** Misses legitimate slow transitions (person pauses)
**Too wide:** False candidates (unrelated tracks matched)

**Approach:**
- Start wide (max_time = 10-15s)
- Learn from validated merges (tighten to typical ± margin)
- Separate windows per portal (some longer than others)

---

### Face Recognition Distance
**TUNING: Cosine distance threshold (0.6? 0.7? 0.8?)**

**Too strict:** Misses same person (lighting change, angle)
**Too loose:** False matches (similar faces)

**Approach:** Calibrate against known validation set

---

## Data Quality Risks

### Face Crop Quality
**HARD: When is face good enough?**

**Challenge:**
- Blurry faces
- Extreme angles (profile, looking down)
- Occlusions (hat, mask, hand)
- Lighting (backlit, shadows)

**Risk:**
- Poor quality crops → bad embeddings → false negatives
- System fails to recognize known people

**Mitigation:**
- Quality scoring (blur, brightness, pose)
- Only register high-quality crops in library
- Multiple crops per person (various angles/lighting)

---

### LLM Panel Quality
**TUNING: Which images to show LLM?**

**Challenge:**
- Track has 50 frames, which to include in panel?
- Best quality? Evenly spaced? Face-visible?

**Risk:**
- Poor images → LLM makes wrong decision
- Too many images → expensive, slow

**Mitigation:**
- Top K quality-scored frames (e.g., K=5)
- Prefer face-visible frames
- Tight crops (minimal background)

---

## Computational Risks

### GPU Budget
**HARD: Limited inference capacity**

**Challenge:**
- 10 cameras × 10 FPS = 100 FPS sustained
- GPU can't process all frames if all cameras Active

**Risk:**
- Queue backlog
- Dropped frames
- Delayed state transitions

**Mitigation:**
- Adaptive inference rate (prioritize Active cameras)
- Suppressed state (reduce sampling under pressure)
- Batch processing across cameras

---

### LLM Cost/Latency
**TUNING: When to call LLM?**

**Challenge:**
- LLM calls are slow (~1-2s) and cost money
- Every merge candidate → expensive
- Too conservative → manual review queue piles up

**Risk:**
- High cost if overused
- Slow timeline construction

**Mitigation:**
- Time/space gating first (filter impossible)
- Face recognition first (auto-merge known people)
- LLM only for ambiguous cases
- Cache results (same track pair)

---

## Learning Risks

---

## Vehicle Association Complexity

### Sporadic Interior Detections
**HARD: Unreliable visibility through windows**

**Challenge:**
- Window glare, tint, angle = intermittent detection
- Driver visible, passenger not (or vice versa)
- Face may be visible one frame, occluded next

**Risk:**
- Miss face match opportunity
- Can't distinguish driver from passenger

**Mitigation:**
- Don't require continuous detection (treat as bonus evidence)
- Any face match during journey → strengthens assignment
- Accept gaps (timeline shows "in vehicle" without continuous track)

---

### Multi-Occupant Exit Assignment
**HARD: Which person exits?**

**Challenge:**
- Two people enter vehicle
- Vehicle moves across cameras
- One person exits → which one?

**Risk:**
- Wrong person assigned to exit track
- Breaks timeline for both agents

**Mitigation:**
- Face match if available (exit track shows face)
- Process of elimination (other agent seen elsewhere)
- Accept uncertainty ("possibly Agent A or B")
- LLM/human review for ambiguous cases

---

### Driver vs Passenger
**MEDIUM: Different exit points**

**Challenge:**
- Driver exits at front, passenger at back
- Spatial position may indicate role
- But swapping possible (passenger becomes driver)

**Risk:**
- Assume driver always front exit → wrong if passenger drove

**Mitigation:**
- Don't assume roles (just track occupancy)
- Let evidence determine assignment (face, timing, elimination)

---

### Face Library Pollution
**HARD: Wrong merge → wrong face registration**

**Challenge:**
- System merges Person A track with Person B track
- Registers Person B face under Person A identity
- Future Person B appearances → matched as Person A

**Risk:**
- Cascading errors (wrong identity propagates)

**Mitigation:**
- Conservative auto-merge thresholds initially
- Human validation before face registration
- Manual admin tool to correct identities

---

## Failure Modes

### ID Swap
**HIGH IMPACT:** Wrong merge breaks timeline
- Person A merged with Person B track
- Timeline shows Person A doing Person B's activity

**Detection:** Timeline review, user correction
**Recovery:** Manual split, re-merge

---

### Missed Merge
**MEDIUM IMPACT:** Timeline fragmented
- Same person appears as multiple unknown agents

**Detection:** User notices duplicate unknowns
**Recovery:** Manual merge

---

### False Promotion
**LOW IMPACT:** Noise promoted to event
- Tree shadow creates track
- Promoted to "unknown agent"

**Detection:** Timeline review
**Recovery:** Mark as noise, exclude from timeline

---

## Related Documents
- **L2 (Strategy):** Why we accept certain tradeoffs
- **L3 (Tactics):** Where these complexities arise
- **L6 (Deployment):** Phased approach to manage risks
- **L11-13 (Technical):** Implementation details
