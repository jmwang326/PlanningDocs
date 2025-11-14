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

## Core Use Case: Security

While real-time alarms are out of scope, the system's ultimate value lies in answering the most critical security question: **Is there an unknown person on the property, and how did they get here?** There is a profound difference between a known person, like a spouse, arriving via a normal entry point and an unidentified individual appearing in the backyard without having passed through a gate. This system is designed to provide that crucial context.

**Primary threat model: Unknown person appears on property.**

**Example scenario:**
- Person climbs over fence (no entry portal)
- Walks across cameras A→B→C
- Enters garage via side door

**What matters:**
- **Who:** Unknown vs known person (face recognition)
- **Where:** Path through property (track merging)
- **When:** Timeline of activity (temporal reconstruction)

**What doesn't matter (out of scope):**
- Real-time alarms (handled by existing motion detection)
- Behavioral predictions ("unusual route")
- Alert fatigue reduction (filtering noise helps, but not the goal)

**This system answers:** "What happened?" not "Should I be notified?"

Forensic reconstruction enables review after-the-fact. Timeline shows complete picture: where unknown person came from, where they went, how long they stayed.

**Note:** Real-time alerting, alarm zones, and notification logic are handled by existing systems (e.g., Blue Iris motion detection). This system focuses on **post-event reconstruction** and **identity tracking**.

## Guiding Principles
The strategy for achieving this mission is guided by a set of core principles, which are detailed in [L2_Strategy_Principles.md](L2_Strategy_Principles.md). In summary, these are:

- **Agent-Centric Thinking**
- **Observable Evidence Only**
- **Learn from Data, Then Refine with Configuration**
- **Count Evidence, Don't Do Arithmetic**
- **Human-in-Loop Initially, Automation via Learning**
- **Acceptable Errors**
- **Learn from Clean Data**

## Goals

### Primary: Unified Agent Timeline
Reconstruct **what happened** from disconnected observations.
- Input: 10 cameras × 10 tracks/day = 100 disconnected observations
- Output: "Person A did X, Person B did Y, Car C did Z"

### Secondary: Smooth Playback
When event identified, saved frames provide smooth playback.
- No missing frames during critical moments
- 25s processing buffer provides historical context (captures lead-up as artifact of chunked processing)
- Post-roll captures complete exit/departure

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
