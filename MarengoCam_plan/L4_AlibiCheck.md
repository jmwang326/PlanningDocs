# Level 4 - Alibi Check (Process of Elimination)

## Purpose
Specification for the alibi check concept - ruling out impossible candidates by verifying they couldn't have reached the entry point at the observed time.

**Referenced from:** [L3_IdentityResolution.md](L3_IdentityResolution.md)

**Implemented in:** L14_AlibiChecker.py

---

## Concept

**Question:** "Could the entry agent have come from elsewhere?"

**Purpose:** Rule out candidates who were seen elsewhere at incompatible times

**Example:**
```
Track_Entry at FrontDoor at T=10:35:00
Candidates: Agent_A (Driveway at T=10:34:48)
            Agent_B (Garage at T=10:34:55)

Alibi check:
- Agent_A: 12s travel (expected 4-6s) - POSSIBLE (may have stopped)
- Agent_B: 5s travel (expected 3-7s) - POSSIBLE

Result: 2 alternatives → AMBIGUOUS
```

---

## Algorithm

**Step 1: Find candidates**
```python
candidates = all_agents_seen_recently(entry_track.start_time - 30s)
```

**Step 2: Check travel time plausibility**
```python
for candidate in candidates:
    travel_time = entry_track.start_time - candidate.end_time
    expected = grid[(candidate.camera, entry_track.camera)]

    if travel_time < expected * 0.5:
        # Too fast, impossible
        continue
    # Accept delays (may have stopped)
    plausible_candidates.add(candidate)
```

**Step 3: Decision**
```python
if len(plausible_candidates) == 0:
    return "DEFINITIVE"  # Must be from current candidate
elif len(plausible_candidates) == 1:
    return "AMBIGUOUS"  # Binary choice
else:
    return "AMBIGUOUS"  # Multi-way choice
```

---

## Cascading Elimination

**When Agent_A is confirmed in track_1, remove Agent_A from all other uncertain tracks at same time:**

```python
def apply_elimination(resolved_track):
    agent_id = resolved_track.global_agent_id

    # Find uncertain tracks at same timestamp
    uncertain_tracks = db.query(LocalTrack).filter(
        overlaps_time(resolved_track),
        global_agent_id == None,
        identity_candidates.contains(agent_id)
    ).all()

    for track in uncertain_tracks:
        track.identity_candidates.discard(agent_id)

        if len(track.identity_candidates) == 1:
            # Only one remains, auto-resolve
            track.global_agent_id = list(track.identity_candidates)[0]
            propagate_resolution(track)
            apply_elimination(track)  # Cascade
```

**Result:** Resolving one track can trigger cascade of resolutions.

---

## Configuration

```yaml
thresholds:
  alibi_check:
    enabled: true
    lookback_window: 30  # seconds
    max_alternative_sources: 2  # Queue if > 2 alternatives
    reject_too_fast_only: true  # Accept delays
```

---

## Edge Cases

**No grid data:** Fall back to face match or review (conservative)

**Portal transitions:** Skip alibi check (time irrelevant)

**Multiple agents, same source:** Queue for review, use face match to disambiguate

---

## Metrics

**Definitive rate:** Target > 60% (after bootstrap)

**Distribution:**
- 0 alternatives: 60% (definitive, auto-merge)
- 1 alternative: 30% (binary choice, LLM review)
- 2+ alternatives: 10% (complex, human review)

---

## Related Documents

**Tactical:** [L3_IdentityResolution.md](L3_IdentityResolution.md), [L3_EvidenceProcessing.md](L3_EvidenceProcessing.md)

**Other L4:** L4_GridLearning.md (provides travel times), L4_PortalTransitions.md (bypasses alibi)

**Algorithms:** L12_ProcessOfElimination.md, L12_AlibiCheck.md

**Implementation:** L14_AlibiChecker.py
