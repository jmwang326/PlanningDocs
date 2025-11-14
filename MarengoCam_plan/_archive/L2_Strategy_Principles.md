# Level 2 - Strategy Principles

## Purpose
This document summarizes the core principles that guide the system's strategy. It is the single source of truth for the "why" behind our architectural decisions.

## Principles

### 1. Agent-Centric Thinking
**Agents are entities** (people, vehicles), not just detections.
- A person exists even when not visible (inside building, under blanket, between cameras)
- Timeline follows the agent, not the camera
- Goal: "What did Person A do?" not "What did Camera A see?"

### 2. Observable Evidence Only
**Use what you can see, not what you assume.**
- No statistical priors ("Person A always drives Car B")
- No routine modeling ("Mail arrives at 3pm")
- Evidence: face match, time/space proximity, visual similarity, portal crossing

**Why:** Assumptions fail on edge cases (visitor drives your car, vacation disrupts routine). Observable evidence is debuggable.

### 3. Learn from Data, Then Refine with Configuration
**The system should first learn the property layout empirically.**
- Let the system discover adjacencies and travel times by observing movement.
- Once the system has a baseline understanding, use configuration to refine it.
- Manually define special zones like portals (doors, gates) to add semantic meaning to learned pathways.

**Why:** This combines the best of both worlds. The system does the heavy lifting of discovering how cameras relate to each other, and the user provides high-level knowledge to correct and enhance that understanding, rather than starting from scratch.

### 4. Count Evidence, Don't Do Arithmetic
**Codifiable and observable.**
- Strong evidence: face match, portal crossing, only candidate
- Weak evidence: spatial proximity, timing fit, similar clothing

**Decision:** Count evidence types, use thresholds.
- ✓ `"Face Match + Plausible Travel Time → auto-merge"`
- ✗ `"Score = 0.30×time + 0.40×spatial + 0.50×visual"`

**Why:** You can't debug weights. You CAN debug "why did face match fail?"

### 5. Human-in-Loop Initially, Automation via Learning
**Trust is earned.**
- Start with human validation (build ground truth)
- System learns from corrections
- Gradually increase automation as confidence builds

**Why:** Mistakes early are costly (wrong merge breaks timeline). Humans teach the system what "same person" looks like.

### 6. Acceptable Errors
**Not all mistakes matter equally.**
- Low impact: "Person in garage" vs "Person in car in garage" → both "inside," self-corrects when person emerges
- High impact: "Person A merged with Person B" → timeline broken, needs correction

**Why:** Over-engineering low-impact edge cases wastes time. Focus precision where it matters.

### 7. Learn from Clean Data
**Garbage in, garbage out.** The system's learned knowledge, especially inter-camera travel times, is only as good as the data it learns from.
- A group of three people walking through a door will have a different travel time than a single person.
- An ambiguous merge that was incorrectly resolved should not pollute the timing data.

**Principle:** Only update core system knowledge (like the `6x6` grid timings) from high-confidence, unambiguous, single-agent observations. Merges involving groups or uncertainty are still valuable for tracking agents, but they are excluded from the learning process.

**Why:** This prevents the system's core understanding of the environment from being skewed by outliers or ambiguous situations, leading to more reliable merge decisions over time.
