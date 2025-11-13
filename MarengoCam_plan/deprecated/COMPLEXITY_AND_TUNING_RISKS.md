# Complexity & Tuning Risks — Critical Review

**Purpose:** Identify all decision logic that is complicated (failure-prone) or tuning-dependent (sensitive to threshold changes). Highlight areas requiring careful observation during rollout.

**Definition of "complicated":** Multiple conditional branches, ambiguous edge cases, non-obvious failure modes, or requires empirical calibration.

---

## 1. Merge Logic — MOST COMPLEX

### Time/Space Filtering (Primary Algorithm)

**Complexity: MODERATE**

**Logic:**
```python
# Step 1: Temporal window
if (track2.start_time - track1.end_time) > 60s:
    reject  # Too long
if (track2.start_time - track1.end_time) < 0 and not overlapping_cameras:
    reject  # Can't merge into the past (unless concurrent)

# Step 2: Spatial plausibility
distance = euclidean(track1.last_fine_grid, track2.first_fine_grid)
expected_distance = walking_speed × time_gap  # walking_speed ~1.0 m/s
if distance > expected_distance × 2.5:
    reject  # Spatially impossible
```

**Tuning dependencies:**
- `max_time_gap = 60s` — Too short misses slow walkers, too long creates false positives
- `walking_speed = 1.0 m/s` — Varies by person (kids slower, adults faster)
- `plausibility_factor = 2.5` — Safety margin for running/rushing

**Failure modes:**
- Person pauses mid-transition (sits on bench) → time gap exceeds window
- Person runs through portal → distance plausible but unusual
- Camera clocks slightly out of sync → temporal ordering wrong

**Mitigation:**
- Wide initial windows (60s generous)
- Re-ID breaks ties when time/space ambiguous
- Human review for edge cases

**Risk level: LOW** — Simple thresholds, failures caught by Re-ID or human review.

---

### Re-ID Disambiguation (When Multiple Candidates)

**Complexity: MODERATE**

**Logic:**
```python
# If time/space narrows to 2-5 candidates:
similarities = [compute_reid_similarity(track1, c) for c in candidates]
best_idx = argmax(similarities)
second_best_idx = argsort(similarities)[-2]

if similarities[best_idx] > 0.85 and (similarities[best_idx] - similarities[second_best_idx]) > 0.15:
    match to candidates[best_idx]
else:
    flag possible_switched_agent, proceed to human/LLM
```

**Tuning dependencies:**
- `reid_threshold = 0.85` — Too high misses correct matches, too low creates false positives
- `confidence_margin = 0.15` — How much better must best be than second-best?

**Failure modes:**
- Identical twins → Re-ID similarities both ~0.92, margin < 0.15
- Poor lighting → all similarities low (~0.60), no clear winner
- Clothing change mid-transition → Re-ID fails, falls back to LLM

**Mitigation:**
- Accept ambiguity with `possible_switched_agent` flag
- LLM fallback for Re-ID failures
- Face recognition overrides Re-ID for known people

**Risk level: MODERATE** — Re-ID threshold calibration critical. Need real-world data to tune.

---

### Multi-Person Handling (2-in-2-out Strategy)

**Complexity: HIGH**

**Logic:**
```python
# Two agents on CameraA, two on CameraB
# Re-ID makes best 1:1 matching (even if mediocre):
matches = greedy_best_match(tracks_A, tracks_B, reid_similarity)

for match in matches:
    if match.similarity < 0.75:  # Low confidence
        match.possible_switched_agent = True
    
    if track1.max_concurrent_agents > 1 or track2.max_concurrent_agents > 1:
        match.possible_switched_agent = True  # Multi-person contamination
```

**Tuning dependencies:**
- `greedy_matching` vs `optimal_assignment` (Hungarian algorithm)
- Threshold for flagging low-confidence matches

**Failure modes:**
- Two people swap positions mid-transition → Re-ID picks wrong pairing
- One person occludes the other → only 1 Re-ID embedding available
- Three people enter, two exit → unbalanced matching

**Mitigation:**
- Flag all multi-person merges as `possible_switched_agent`
- Exclude from portal timing learning
- Human review corrects mismatches
- False merge recoverable (split and re-merge)

**Risk level: HIGH** — Most complex scenario. Requires extensive real-world testing. But consequences limited (flagged for review, doesn't contaminate learning).

---

### Solo Track Learning Filter

**Complexity: LOW**

**Logic:**
```python
learn_portal_timing = (
    track1.solo_track and 
    track2.solo_track and 
    not merge.possible_switched_agent and
    merge.validated
)
```

**Tuning dependencies:** None (boolean flags)

**Failure modes:**
- Detector misses second person → track falsely marked `solo_track=True`
- Two people walk far apart → both marked solo, but behavior different

**Mitigation:**
- Conservative solo detection: if ANY frame has >1 detection, mark multi-person
- Portal timing window starts wide, only tightens with 50+ clean observations

**Risk level: LOW** — Simple boolean filter, worst case is slower learning (no corruption).

---

## 2. Portal Timing — MODERATE COMPLEXITY

### Portal Window Tightening (Phase 4 Optional)

**Complexity: MODERATE**

**Logic:**
```python
# After 50+ validated observations:
observations = get_portal_observations(portal_id, validated=True, solo_only=True)
p50 = percentile(observations, 50)
p90 = percentile(observations, 90)
std = std_dev(observations)

new_max_time = min(p90 + 2*std, configured_max_time)  # Never exceed user config
```

**Tuning dependencies:**
- `min_observations = 50` — Too few leads to overfit, too many delays learning
- `P90 + 2*std` — Safety margin formula (arbitrary choice)

**Failure modes:**
- Outlier observation (person pauses to tie shoe) → widens P90
- Seasonal changes (winter: slower, summer: faster) → stats drift
- Portal used by kids vs adults → bimodal distribution, P90 misleading

**Mitigation:**
- User-configured `max_time` is hard ceiling (never violated)
- Rolling window (last 100 observations) adapts to changes
- Only tighten, never loosen automatically (prevents runaway)

**Risk level: MODERATE** — Statistics-based, but user override prevents catastrophic failure. Worst case: portal timing stays wide (more LLM calls, no correctness impact).

---

## 3. Face Recognition — LOW COMPLEXITY

### Face Match Thresholds

**Complexity: LOW**

**Logic:**
```python
similarity = cosine_similarity(face1, face2)

if similarity >= 0.60:
    return AUTO_MERGE  # High confidence match
elif similarity <= 0.40:
    return NO_MERGE    # Hard contradiction
else:
    return ASK_LLM     # Uncertain
```

**Tuning dependencies:**
- `auto_merge_threshold = 0.60` — Too low creates false positives
- `reject_threshold = 0.40` — Too high rejects correct matches

**Failure modes:**
- Face at extreme angle → low similarity despite same person
- Poor lighting → embedding quality degraded
- Sunglasses/mask → face detection fails entirely

**Mitigation:**
- CodeProject.AI provides quality scores → filter low-quality embeddings
- LLM fallback for uncertain cases
- Multiple face crops per track → use best quality

**Risk level: LOW** — Well-understood computer vision problem. CodeProject.AI provides calibrated thresholds out-of-box.

---

## 4. LLM Adjudication — LOW COMPLEXITY (HUMAN OVERRIDE)

### Confidence Acceptance

**Complexity: LOW**

**Logic:**
```python
if llm.answer == "YES" and llm.confidence >= 0.80:
    return ACCEPT
elif llm.answer == "NO" and llm.confidence >= 0.70:
    return REJECT
else:
    return HUMAN_REVIEW  # Uncertain
```

**Tuning dependencies:**
- `yes_threshold = 0.80` — Calibrate against human agreement rate
- `no_threshold = 0.70` — Asymmetric (rejecting safer than accepting)

**Failure modes:**
- LLM overconfident on wrong answer → false auto-merge
- LLM underconfident on obvious match → unnecessary human review

**Mitigation:**
- **Human-first rollout:** Week 1-2 = 100% human review, no LLM auto-accept
- Track LLM accuracy vs human decisions
- Only enable auto-accept after demonstrated accuracy (Week 6+)
- Start conservative (confidence > 0.95), lower gradually

**Risk level: LOW** — Human override prevents catastrophic failure. LLM is advisory until proven.

---

## 5. Camera State Transitions — LOW COMPLEXITY

### State Transition Timing

**Complexity: LOW**

**Logic:**
```python
# Standby → Armed
if detection.confidence > 0.55 and detection.class in ['person', 'vehicle']:
    transition to Armed

# Armed → Active
if track.duration >= 3.0s:
    transition to Active

# Active → Post
if no_detections_for(quiet_window_s=5.0):
    transition to Post

# Post → Standby
after post_roll_s=10.0:
    transition to Standby
```

**Tuning dependencies:**
- `sustained_duration = 3.0s` — Too short = false alarms, too long = missed events
- `quiet_window_s = 5.0s` — Occlusion tolerance
- `post_roll_s = 10.0s` — Capture departure

**Failure modes:**
- Person hides under blanket for 6s → camera drops to Standby, misses emergence
- Fast-moving person crosses frame in 2s → never reaches Active

**Mitigation:**
- Armed state at 3 FPS provides rapid confirmation before Active
- Post state maintains 10 FPS for 10s (catches brief occlusions)
- Tuning values observable in logs, easy to adjust

**Risk level: LOW** — Simple timers, failures are visible in footage, easy to tune.

---

## 6. Height Calibration — LOW COMPLEXITY

### Online Learning Convergence

**Complexity: LOW**

**Logic:**
```python
# Weighted average for each grid cell:
new_mean = (1 - alpha) * old_mean + alpha * observed_height
alpha = 0.2  # Learning rate
```

**Tuning dependencies:**
- `alpha = 0.2` — Too high is noisy, too low is slow

**Failure modes:**
- Kids vs adults → cell has bimodal height distribution
- Camera moved/adjusted → invalidates entire grid

**Mitigation:**
- Only used for depth normalization, NOT for ID swap detection
- Convergence observable (plot mean ± std per cell)
- Camera adjustment detected by sharp mean change → reset grid

**Risk level: LOW** — Not critical to correctness. Worst case: less accurate depth normalization.

---

## Summary Risk Assessment

### HIGH RISK (Requires Extensive Testing)
1. **Multi-person merge matching** — 2-in-2-out scenarios, identity swaps
   - Mitigation: Flag all multi-person as uncertain, human review required
   - Worst case: Flagged merge is incorrect, human corrects later

### MODERATE RISK (Requires Calibration)
2. **Re-ID threshold tuning** — Balance false positives vs false negatives
   - Mitigation: LLM fallback for ambiguous cases
   - Needs real-world data to calibrate

3. **Portal timing statistics** — P50/P90 estimation, outlier handling
   - Mitigation: User-configured max_time hard ceiling
   - Only affects efficiency (LLM call volume), not correctness

4. **Time/space plausibility factors** — Walking speed, safety margins
   - Mitigation: Wide initial windows, tighten conservatively
   - Re-ID catches misses from overly tight windows

### LOW RISK (Well-Understood or Human-Supervised)
5. **Face recognition thresholds** — Mature CV problem, vendor-calibrated
6. **LLM confidence** — Human override during learning phase
7. **Camera state timers** — Observable in logs, easy to tune
8. **Height calibration** — Not critical to correctness
9. **Solo track filter** — Simple boolean, worst case is slower learning

---

## Recommended Testing Strategy

### Phase 2 Focus Areas (Highest Risk First)

1. **Multi-person scenarios:**
   - 2 people walking together through portal
   - 2 people crossing paths (swap positions)
   - 3 people entering, 2 exiting (unbalanced)
   - **Goal:** Verify `possible_switched_agent` flag catches ambiguity

2. **Re-ID threshold calibration:**
   - Collect 100+ validated merges
   - Plot similarity distribution for correct vs incorrect matches
   - Find threshold that maximizes accuracy
   - **Goal:** Confidence > 0.85 = >95% correct

3. **Time/space edge cases:**
   - Person pauses mid-transition (sit on bench 30s)
   - Person runs through portal (2× walking speed)
   - Person takes very slow path (kid wandering)
   - **Goal:** Wide window (60s) catches all reasonable cases

4. **Portal timing learning:**
   - Verify solo track filter excludes multi-person
   - Check P50/P90 estimation after 50 observations
   - Test outlier handling (one 45s observation in 50× 2s observations)
   - **Goal:** Portal window tightens without false rejections

5. **LLM accuracy tracking:**
   - Week 1: 100% human review, log LLM suggestions
   - Calculate agreement rate (% LLM matches human decision)
   - **Goal:** >90% agreement before enabling auto-accept

### Tuning Dials (In Priority Order)

**Critical (Tune First):**
1. Re-ID similarity threshold (0.85 baseline)
2. Re-ID confidence margin (0.15 baseline)
3. Time/space walking speed (1.0 m/s baseline)
4. LLM confidence thresholds (0.80 YES, 0.70 NO)

**Important (Tune After 1 Week):**
5. Portal timing min_observations (50 baseline)
6. Camera state sustained_duration (3.0s baseline)
7. Face recognition auto_merge threshold (0.60 baseline)

**Optional (Tune After 1 Month):**
8. Portal P90 safety margin (2*std baseline)
9. Height calibration learning rate (0.2 baseline)
10. Post-roll timing (10s baseline)

---

## Decision: What to Document vs What to Monitor

### Document Now (Architecture-Level):
- ✅ Unified concurrent matching algorithm
- ✅ Solo track learning filter
- ✅ Multi-person 2-in-2-out strategy
- ✅ Human-first LLM rollout plan
- ✅ Dual-scale grid rationale

### Monitor During Rollout (Empirical):
- ⏳ Re-ID threshold calibration (needs real data)
- ⏳ Portal timing statistics convergence (needs observations)
- ⏳ LLM accuracy tracking (needs human validation dataset)
- ⏳ Multi-person scenario handling (needs live testing)

---

## Conclusion

**Portal timing is NOT the most complex area.** It's actually simple statistics with user override. The real complexity is:

1. **Multi-person merge matching** (highest risk, but mitigated by uncertainty flags)
2. **Re-ID threshold calibration** (moderate risk, needs empirical data)
3. **Time/space plausibility filters** (moderate risk, wide windows reduce failures)

All three have **good failure modes:**
- Multi-person: Flagged for review, doesn't contaminate learning
- Re-ID: Falls back to LLM or human
- Time/space: Re-ID catches misses, LLM is safety net

**No catastrophic failure paths.** Worst case is more human review (slower automation), not silent corruption of learned data.

**Recommendation:** Proceed with documented architecture. Focus testing on multi-person scenarios and Re-ID calibration during Phase 2 rollout.

---

**Last updated:** 2025-11-08
