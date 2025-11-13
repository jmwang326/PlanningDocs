# Level 1 - Mission

## Core Problem

**Camera-by-camera systems flood you with useless alerts.**
- 100 motion alerts per day
- 95 are noise (trees, shadows, cars passing by)
- 5 are real activity buried in the noise
- You miss what matters

Multi-camera surveillance produces **disconnected observations**:
- Camera A sees a person walking (Track 1)
- Camera B sees a person walking (Track 2)
- **Question:** Are these the same person? Is this real activity or noise?

**Without solving this:** Timeline is fragmented. Alerts are overwhelming. Manual correlation required.

**With solution:** Unified agent timeline. Filter noise. See what happened: "Person A entered property, walked through Cameras A→B→C, exited via gate."

## The Assignment Problem

**Fundamentally:** Track detection and merging.

1. **Detect tracks** within each camera (YOLO11 provides tracking)
2. **Merge tracks** across cameras when evidence suggests same agent
3. **Handle uncertainty** when evidence is ambiguous or missing
   - Tracks may belong to multiple possible agents
   - Uncertainty persists until resolved by later evidence
   - System operates with partial information

**Modern toolkit available:**
- YOLO11 (object detection + tracking)
- Face recognition (identify known people)
- LLMs (adjudicate ambiguous cases)
- Human-in-loop (validate and teach system)

## Core Values

### Agent-Centric Thinking
**Agents are entities** (people, vehicles), not just detections.
- A person exists even when not visible (inside building, under blanket, between cameras)
- Timeline follows the agent, not the camera
- Goal: "What did Person A do?" not "What did Camera A see?"

### Observable Evidence Only
**Use what you can see, not what you assume.**
- No statistical priors ("Person A always drives Car B")
- No routine modeling ("Mail arrives at 3pm")
- Evidence: face match, time/space proximity, visual similarity, portal crossing

**Why:** Assumptions fail on edge cases (visitor drives your car, vacation disrupts routine). Observable evidence is debuggable.

### Configuration Over Blind Learning
**User knows the property layout.**
- Manually configure portals (doors, gates, exits)
- Let system learn timing (how long to walk through door?)
- Don't rediscover what's already known

**Why:** User has ground truth. Statistical discovery of "doors exist" is wasteful.

### Count Evidence, Don't Do Arithmetic
**Codifiable and observable.**
- Strong evidence: face match, portal crossing, only candidate
- Weak evidence: spatial proximity, timing fit, similar clothing

**Decision:** Count evidence types, use thresholds.
- ✓ "Strong evidence >= 2 → auto-merge"
- ✗ "Score = 0.30×time + 0.40×spatial + 0.50×visual"

**Why:** Can't debug weights. CAN debug "why did face match fail?"

### Human-in-Loop Initially, Automation via Learning
**Trust is earned.**
- Start with human validation (build ground truth)
- System learns from corrections
- Gradually increase automation as confidence builds

**Why:** Mistakes early are costly (wrong merge breaks timeline). Humans teach system what "same person" looks like.

### Acceptable Errors
**Not all mistakes matter equally.**
- Low impact: "Person in garage" vs "Person in car in garage" → both "inside," self-corrects when person emerges
- High impact: "Person A merged with Person B" → timeline broken, needs correction

**Why:** Over-engineering low-impact edge cases wastes time. Focus precision where it matters.

## Goals

### Primary: Unified Agent Timeline
Reconstruct **what happened** from disconnected observations.
- Input: 10 cameras × 10 tracks/day = 100 disconnected observations
- Output: "Person A did X, Person B did Y, Car C did Z"

### Secondary: Smooth Playback
When event identified, saved frames provide smooth playback.
- No missing frames during critical moments
- Pre-roll captures lead-up
- Post-roll captures exit

### Tertiary: Known Identity Recognition
After weeks of operation, auto-identify regular visitors.
- Face library builds from validated merges
- Known people auto-merge without human review
- Unknown people/vehicles still require validation

## What This System Is NOT

### Not Real-Time Alerting
**This is forensic reconstruction.**
- Inference may lag acquisition
- Merge decisions happen periodically (not every frame)
- Timeline built retroactively

**For alerts:** Use existing systems (Blue Iris motion detection).

### Not Behavior Modeling
**No predictive analytics.**
- Don't model "Person A usually arrives at 5pm"
- Don't flag "unusual activity"
- Observable evidence only

### Not Cloud-Dependent
**Everything runs locally** (except optional LLM API calls).
- Privacy: footage stays on-premises
- No internet dependency for core functionality
- Cost: no cloud storage fees

## Success Definition

**Phase-by-phase criteria exist in L6 (Deployment Plan).**

**High-level success:**
- Agent timeline accurately reconstructs activity
- Known people auto-identified without manual review
- System runs with minimal human intervention
- False merge rate < 2%

## Terminology Reference
See [L10_Nomenclature.md](L10_Nomenclature.md) for standard terminology (agent, track, portal, merge, evidence).

## Related Documents
- **L2 (Strategy):** How we solve the assignment problem (non-technical approach)
- **L6 (Deployment):** Phased rollout plan and success criteria per phase
- **L10 (Nomenclature):** Terminology definitions
