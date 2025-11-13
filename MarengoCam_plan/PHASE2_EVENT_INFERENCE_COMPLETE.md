# Event Inference - Phase 2 Implementation

## Overview
Phase 2 implements intelligent event detection using a state machine that decides when to START and END recording based on detection patterns, not just single motion spikes.

## Status: ✅ COMPLETE

### Components Created

1. **`src/events/camera_state.py`** - Camera State Machine
   - States: STANDBY → ARMED → ACTIVE → POST
   - Adaptive inference rates per state
   - 3-second sustained rule (not just single spikes)
   - Configurable thresholds and timings

2. **`dev_tools/test_event_inference.py`** - Test Suite
   - 4 scenarios demonstrating event logic
   - Person walkby (creates event)
   - Brief motion (no event - too short)
   - Car parking (long event with tail)
   - False alarm (armed expires)

3. **Integration with `frame_loop.py`**
   - State machine per camera
   - Tracks state transitions
   - Counts events created
   - Logs transitions to console

## How It Works

### State Transitions

```
STANDBY (0.1 FPS inference)
    ↓ detection > θ_arm
ARMED (0.3 FPS inference) - watching for confirmation
    ↓ sustained 3s OR high confidence (θ_confirm)
ACTIVE (1.0 FPS inference) - recording event
    ↓ quiet for quiet_window_s
POST (1.0 FPS inference) - capturing tail
    ↓ post_roll_s complete
STANDBY
```

### Key Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `theta_arm` | 0.4 | Min confidence to enter ARMED |
| `theta_active` | 0.5 | Min confidence to stay ACTIVE |
| `theta_confirm` | 0.7 | High confidence for instant promotion |
| `sustained_time_s` | 3.0 | How long detections must persist |
| `gap_tolerance_s` | 0.3 | Allow small gaps in sustained tracks |
| `arm_expire_s` | 5.0 | ARMED → STANDBY if no confirmation |
| `quiet_window_s` | 2.5 | ACTIVE → POST when quiet |
| `post_roll_s` | 12.0 | Duration of POST before STANDBY |

### 3-Second Sustained Rule

**Problem:** Single detection spikes create false events (birds, shadows, headlight reflections)

**Solution:** Require detections to persist for 3+ seconds with small gap tolerance

**Implementation:**
- Track all detections in rolling history (last 100)
- Group into segments (gap ≤ 0.3s = same segment)
- Sum segment durations
- Promote to ACTIVE when total ≥ 3.0s

**Example:**
```
Detections at: 1.0s, 1.1s, 1.2s, 1.5s, 2.0s, 2.1s, 3.5s, 3.6s, 4.0s
Segments: [1.0-1.2s], [1.5s], [2.0-2.1s], [3.5-3.6s], [4.0s]
         (0.2s)      (0.0s)   (0.1s)      (0.1s)      (0.0s)
Total: 0.4s (too short - stay ARMED)

Detections at: 1.0s, 1.1s, 1.2s, 1.3s, 1.4s, ..., 3.8s, 3.9s, 4.0s
Segments: [1.0-4.0s]
Total: 3.0s (PROMOTE to ACTIVE)
```

### Inference Rate Scheduling

State determines how often frames are sent to YOLO:

- **STANDBY (0.1):** 1 in 10 frames (saves GPU, just watching)
- **ARMED (0.3):** 3 in 10 frames (faster confirmation)
- **ACTIVE (1.0):** All frames (full fidelity)
- **POST (1.0):** All frames (capture complete exit)

**GPU Budget:**
With 3 cameras at 10 FPS each:
- All STANDBY: 3 frames/sec (0.3 FPS effective)
- All ACTIVE: 30 frames/sec (3.0 FPS effective)
- Mixed: Scales between 0.3-3.0 FPS

Current GPU: 82ms inference = 12.1 FPS max
→ Can handle 3 ACTIVE cameras simultaneously

New GPU (coming): Should handle 10+ ACTIVE cameras

## Test Results

### Scenario 1: Person Walkby ✅
```
STANDBY (2.0s no activity)
  ↓ person conf=0.45
ARMED (watching)
  ↓ sustained 3.5s
ACTIVE (recording)
  ↓ quiet 2.6s
POST (tail capture)
```
**Result:** Event created successfully

### Scenario 2: Brief Motion ❌ (by design)
```
STANDBY
  ↓ bird conf=0.45
ARMED
  ↓ only 0.6s detections (too short)
  ↓ armed expired after 5s
STANDBY
```
**Result:** No event (too brief - correct behavior)

### Scenario 3: Car Parking ✅
```
STANDBY
  ↓ car conf=0.75 (instant high confidence)
ARMED → ACTIVE (instant promotion)
  ↓ 9 seconds of movement
  ↓ quiet 2.6s
POST
  ↓ 12s tail capture
STANDBY
```
**Result:** Full event captured with tail

### Scenario 4: False Alarm ❌ (by design)
```
STANDBY
  ↓ unknown conf=0.42
ARMED
  ↓ intermittent detections (not sustained)
  ↓ armed expired after 5s
STANDBY
```
**Result:** No event (not confirmed - correct behavior)

## Integration Status

### Completed ✅
- State machine implemented
- Test suite validates all scenarios
- Integrated into frame_loop.py
- Logging and stats tracking

### Next Steps (Phase 2 Continuation)

1. **Event Storage** (not started)
   - Create event records when entering ACTIVE
   - Save event metadata (start time, camera, trigger)
   - Link frames to events
   - Manifest generation

2. **Pre-roll Ring Buffer** (not started)
   - Keep last 7 seconds of frames in memory
   - Include in event when entering ACTIVE
   - Captures approach/entry

3. **Cross-Camera Event Grouping** (Phase 3)
   - Multi-camera events (person across cameras)
   - Adjacency-aware grouping
   - Travel time matrix integration

4. **Clip Creation** (Phase 3)
   - MP4 generation from event frames
   - Manifest-based reconstruction
   - Timeline viewer integration

## Configuration

State machine parameters will move to config file:

```yaml
# config/events.yaml
event_detection:
  thresholds:
    arm: 0.4        # Initial detection
    active: 0.5     # Sustained threshold
    confirm: 0.7    # Instant promotion
  
  timing:
    sustained_s: 3.0      # Required persistence
    gap_tolerance_s: 0.3  # Small gaps allowed
    arm_expire_s: 5.0     # Armed timeout
    quiet_window_s: 2.5   # Quiet → POST
    post_roll_s: 12.0     # POST duration
  
  inference_rates:
    standby: 0.1    # 1 in 10 frames
    armed: 0.3      # 3 in 10 frames
    active: 1.0     # All frames
    post: 1.0       # All frames
```

## Performance Impact

### Before (Phase 1)
- All cameras: 1.0 FPS inference (fixed rate)
- No event detection
- No intelligent recording decisions

### After (Phase 2)
- Dynamic inference rate based on state
- STANDBY: 0.1 FPS (90% GPU savings)
- ACTIVE: 1.0 FPS (full fidelity when needed)
- Smart event creation (no false alarms)

### GPU Budget Example
3 cameras, mixed states:
- Camera 1: STANDBY (0.1 FPS)
- Camera 2: ARMED (0.3 FPS)
- Camera 3: ACTIVE (1.0 FPS)

Total: 1.4 FPS (vs 3.0 FPS if all at 1.0 FPS)
**53% GPU savings** while maintaining full fidelity for active events

## Known Limitations

1. **Single-camera only** - Cross-camera grouping is Phase 3
2. **No event persistence** - Events exist only in memory
3. **No pre-roll** - Captures from ARMED onwards, not before
4. **Fixed parameters** - Not yet moved to config file

## Files Modified

- `src/events/camera_state.py` - NEW (449 lines)
- `src/acquisition/frame_loop.py` - Modified (added state machine)
- `dev_tools/test_event_inference.py` - NEW (test suite)

## Testing

```bash
# Test state machine in isolation
python src/events/camera_state.py

# Test with simulated scenarios
python dev_tools/test_event_inference.py

# Test with live cameras (when ready)
python src/acquisition/frame_loop.py
```

## Success Criteria ✅

- [x] State machine handles all transitions correctly
- [x] 3-second sustained rule prevents false alarms
- [x] High-confidence instant promotion works
- [x] Armed expiration prevents lingering armed state
- [x] POST captures complete exits
- [x] Adaptive inference rates save GPU budget
- [x] Test suite validates all scenarios
- [x] Integration with frame_loop.py complete

## Next Session

When new GPU arrives:
1. Run live test with cameras
2. Validate GPU budget with multiple ACTIVE cameras
3. Add event storage and manifest generation
4. Build pre-roll ring buffer
5. Begin Phase 3: Cross-camera grouping
