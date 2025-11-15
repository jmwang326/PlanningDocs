# Level 1 - Mission

## Core Problem

Multi-camera surveillance produces **disconnected observations**:
- Camera A: Track 1 (person walking)
- Camera B: Track 2 (person walking)
- **Question:** Same person? Real activity or noise?

**Without solution:** Fragmented timeline, overwhelming alerts, manual correlation.

**With solution:** Unified agent timeline: "Person A entered via driveway, walked through cameras A→B→C, exited via gate."

## The Assignment Problem

**Detect tracks** within each camera (YOLO11), **merge tracks** across cameras (evidence-based), **handle uncertainty** (multi-assignment, process of elimination).

**Toolkit:**
- YOLO11 (detection + tracking)
- Face recognition (known people)
- LLMs (adjudicate uncertain cases)
- Human-in-loop (bootstrap)

## Primary Use Case

**Answer:** "Is there an unknown person on the property, and how did they get here?"

**What matters:**
- **Who:** Unknown vs known (face recognition)
- **Where:** Path through property (track merging)
- **When:** Timeline (temporal reconstruction)

**Out of scope:**
- Real-time alarms (use Blue Iris motion detection)
- Behavioral predictions
- Cloud storage

**System type:** Forensic reconstruction, not real-time alerting.

## Goals

**Primary:** Unified agent timeline from disconnected observations

**Secondary:** Smooth playback (no missing frames, 25s buffer, post-roll)

**Tertiary:** Known identity auto-recognition (face library)

## Success Criteria

- Timeline accurately reconstructs activity
- Known people auto-identified
- False merge rate < 2%
- Minimal human intervention (80%+ auto-merge after bootstrap)

## Principles

See [L2_Strategy_Principles.md](L2_Strategy_Principles.md): Agent-centric thinking, observable evidence only, learn from clean data, human-in-loop initially, count evidence (don't do arithmetic).
