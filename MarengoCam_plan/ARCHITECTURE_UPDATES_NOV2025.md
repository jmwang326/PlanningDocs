# Architecture Updates - November 2025

This document summarizes the key architectural decisions and changes agreed upon during the November 2025 design review.

## Summary of Changes

The architecture has been refined from the original "detection-centric" approach to an **"agent-centric persistent tracking"** model with a dual-grid system.

## Core Architectural Decisions

### 1. Dual-Grid System

**32×32 Intra-Camera Grid:**
- **Purpose:** Track agent movement and prevent ID swaps within a single camera
- **Use:** Pixel-level tracking with grid-based spatial consistency checks
- **Uniform grid:** Fixed 32×32 cells, each cell = frame_width/32 × frame_height/32 pixels
- **Auto-calibration:** Learn depth from observed person bbox heights per cell
- **Anti-swap:** Spatial continuity + motion vector consistency (not height, too jittery)
- **Flag uncertain:** Mark tracks as `possibly_switched` when swap suspected

**Why uniform grid:**
- Simple, stable - doesn't change if camera moves slightly
- Height calibration per cell handles depth/perspective differences
- Changing grid parameters later would invalidate all learned data (heights, portal cells)
- 32×32 = 1024 cells gives sufficient spatial resolution

**8×8 Inter-Camera Grid:**
- **Purpose:** Learn adjacency between cameras and correlate cross-camera activity
- **Mapping:** Clean 4:1 downsampling from 32×32 (each 4×4 block → 1 cell)
- **Tractability:** 64×64 = 4K edges per camera pair vs. 1024×1024 = 1M edges
- **Learning:** Only from validated single-agent observations
- **Overlaps:** Can be discovered anywhere (not just borders), learned from data

### 2. Agent Persistence Model

**Agents vs. Detections:**
- Agents are persistent entities tracked across cameras and time
- Track segments = where we can SEE the agent (camera-observable windows)
- Agents continue conceptually even when not visible
- Multi-state location tracking with confidence scores when uncertain

**Agent States:**
```python
Agent.current_state = {
    'visible': True/False,
    'camera': 'CamC' / None,
    'cell_32x32': (15, 20) / None,
    'cell_8x8': (3, 5) / None,  # Derived
    'last_seen': timestamp,
    'possibly_switched': False  # ID swap flag
}

# When not visible:
Agent.possible_locations = [
    {'state': 'in_vehicle', 'vehicle': 'V1', 'confidence': 0.70},
    {'state': 'in_structure', 'location': 'garage', 'confidence': 0.25},
    {'state': 'unknown', 'confidence': 0.05}
]
```

### 3. Observable Evidence Only

**Security Analyst Perspective:**
- No statistical ownership priors
- No routine assumptions ("Person A always drives")
- Only what can be SEEN or INFERRED from visible evidence
- Learning adjusts confidence thresholds, not creates priors

**Evidence hierarchy:**
1. Time/space gating (8×8 learned edges)
2. Face match (if available and quality sufficient)
3. Spatial proximity + timing
4. Process of elimination
5. LLM adjudication (Gemini Flash 2.0, cheap)
6. Human review (final fallback)

### 4. State Machine: Per-Camera, Agent-Driven

**Camera states determined by agents in view:**
- **Standby:** Quiet baseline, sparse inference (~10% of 10 FPS)
- **Armed/Alerted:** Recent activity or neighbor Active, medium inference (~60%)
- **Active:** Agent confirmed (≥3s movement), high inference (100%)
- **Post:** Cool-down, medium inference

**Key insight:** If person goes under blanket for 2 minutes:
- Camera stays Active (searching)
- Degrades to Armed/Alerted
- Eventually Standby
- Agent marked with possible_locations (hidden, possibly moved)

### 5. Merge Decision Hierarchy

**Three-tier decision making:**

**Tier 1: Auto-merge (confidence ≥ 0.80)**
- Known overlap edges (Δt ≈ 0, validated)
- Strong face match (>0.85 similarity)
- Only candidate (no ambiguity)
- Time/space fit within tight learned windows

**Tier 2: Suggest + Queue (0.50 ≤ confidence < 0.80)**
- Multiple candidates (≤3)
- Medium evidence strength
- Pre-fill review UI, allow human edit
- Learn from corrections

**Tier 3: Mark uncertain (confidence < 0.50)**
- Too many candidates (>3)
- Weak evidence
- Mark agent with possible_locations
- Revisit when more evidence emerges

### 6. Overlap Discovery Protocol

**Only learn from validated single-agent observations:**
```python
# When is an overlap edge created?
1. Detect near-simultaneous activity on CamA and CamB (Δt < 0.5s)
2. ONLY when single agent visible (no ambiguity)
3. Validate identity across full path (face/appearance confirms same person)
4. Record edge: CamA[cell_8x8] ↔ CamB[cell_8x8], travel_time ≈ 0
5. Future: Auto-merge when same cells occupied simultaneously
```

**Key:** Overlaps can occur anywhere in frame, not just borders. Discovered from data, not assumed from geometry.

### 7. Vehicle Container Handling

**Pragmatic approach with tolerance for errors:**

**Easy case (1 person, 1 vehicle):**
- Auto-merge with high confidence (0.95)

**Hard case (2+ people, 1 vehicle):**
- Score each person using observable evidence:
  - Recency (last to enter)
  - Spatial proximity (near vehicle bay)
  - Face in windshield (if visible)
- Auto-merge if score ≥ 0.80
- Suggest + queue if 0.50 ≤ score < 0.80
- Mark uncertain if score < 0.50

**Uncertainty resolution:**
- Person exits on foot later → rules out vehicle association
- Cascades: Ruling out Person B → increases confidence for Person A

**Acceptable errors:**
- "In garage" vs "in car" (both "inside structure") - low impact
- Will self-correct when person emerges

### 8. Image Bank for Re-ID

**Store best quality images per agent:**
```python
image_bank[agent][camera][lighting_condition] = [top_3_images]
```

**Purpose:**
- Even if face recognition fails in real-time
- Enables manual or batch re-ID days/weeks later
- Diverse lighting conditions improve robustness

**Quality scoring:**
- Face visibility and sharpness
- Brightness and contrast
- Occlusion level
- Bbox size (distance from camera)

### 9. Learning from Corrections

**When human overrides merge decision:**
- Record full context (evidence used, confidence scores)
- Extract learning signals:
  - False positive merge → tighten thresholds
  - False negative → relax thresholds
  - Timing errors → adjust edge windows
- Adjust confidence calibration, not create hard rules

**Example:**
```python
if original == 'MERGE' and override == 'REJECT':
    if 'face_match' in evidence:
        # Face match wasn't sufficient
        face_match_threshold += 0.02
    if delta_t > typical_time * 1.5:
        # Time fit was marginal
        edge_window_factor *= 0.9
```

### 10. Storage & Retention

**Frames (9 days):**
- All sub frames (10 FPS) for Armed/Active/Post cameras
- Main frames during Active (optional, higher resolution)
- Smooth 10 FPS playback reconstruction

**Metadata (indefinite):**
- Agent tracks and segments
- Merge decisions with evidence
- Edge attributes (8×8 inter-grid)
- Corrections and learning signals
- Image banks

**Height calibration (indefinite):**
- Per camera × cell_32x32
- Median bbox height when agent's feet in that cell
- Updated online from observations
- Cold start: assume uniform, refine quickly

### 11. Real-Time Architecture with Elastic Lag

**Tiered latency:**
```
Frame ingestion:        10 FPS, hard real-time
  ↓
Detection + Tracking:   1s lag default, elastic to 5s under load
  ↓
Merge decisions:        2-10s lag, queued
  ↓
LLM adjudication:       Async, 30s+ lag acceptable
  ↓
Human review:           Batched, hours/days lag acceptable
```

**Key:** Critical path (detection, tracking) stays near real-time. High-level decisions can lag without impacting core functionality.

### 12. Portal/Structure Semantics (Deferred)

**Decision:** Start without portal/structure config, add later if needed.

**Phase 1:**
- Pure camera-to-camera adjacency (8×8 edges)
- Long gaps marked as uncertain
- Learn what connects naturally from data

**Phase 2 (if needed):**
- Add Door/Garage/Exit markup
- Structure-level reasoning (house interior as aggregate state)
- Transit time estimates per structure

**Rationale:** Prove basic approach works before adding configuration complexity.

## Implementation Priority

### Phase 0: Foundation
- JPEG acquisition at 10 FPS from Blue Iris
- 32×32 grid overlay and cell mapping
- Height calibration (cold start: uniform, then learn)
- Frame storage and manifest generation

### Phase 1: Single-Camera Tracking
- YOLO detection on sub frames
- Intra-camera tracking with 32×32 grid
- Motion calculation with depth normalization
- Anti-swap detection (flag possibly_switched)
- Image bank storage

### Phase 2: Cross-Camera Correlation
- 8×8 inter-grid downsampling
- Edge discovery (validated single-agent only)
- Time/space merge gating
- Observable evidence scoring
- Auto-merge for high confidence

### Phase 3: Adjudication & Learning
- LLM integration (Gemini Flash 2.0)
- Human review UI
- Correction recording
- Confidence calibration
- Learning feedback loop

### Phase 4: Advanced Features
- Vehicle container handling
- Multi-state location tracking
- Uncertainty collapse
- Re-ID from image banks
- Portal/structure semantics (if needed)

## Key Principles

1. **Agent-centric:** Track people/vehicles conceptually, not just detections
2. **Dual grids:** 32×32 for intra-camera, 8×8 for inter-camera (tractable learning)
3. **Observable only:** No priors, no routines, only visible evidence
4. **Honest uncertainty:** Multi-state tracking when ambiguous, don't force guesses
5. **Self-correcting:** Later observations collapse uncertainties
6. **Learning flywheel:** 50% auto initially → 90%+ after weeks
7. **Pragmatic errors:** Some mistakes acceptable if low-impact and self-correcting
8. **Graceful degradation:** System works even if face/LLM unavailable

## Open Questions / TBD

- Exact confidence thresholds (will calibrate from real data)
- Day/night motion gate differences
- Quiet period timings (state transitions)
- BI main/sub frame selection convention
- When to save main frames for neighbor cameras
- Portal/structure config format (if Phase 2 needed)

---

**Status:** Architecture agreed, ready for implementation planning.

**Date:** November 7, 2025

**Next steps:** Begin Phase 0 foundation work, validate with real camera feeds.
