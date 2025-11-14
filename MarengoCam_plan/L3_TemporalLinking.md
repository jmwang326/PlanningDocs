# Level 3 - Temporal Linking (TemporalLinker)

## Purpose
Tactical guide for linking LocalTracks across chunk boundaries (same camera). This creates continuous observation despite YOLO reassigning IDs each chunk.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for TemporalLinker architecture overview.

---

## Input/Output

**Input:**
- New chunk's LocalTracks (just created by ChunkProcessor)
- Previous chunk's LocalTracks (from database, filtered to overlap zone)

**Output:** Updated previous_track/next_track pointers on both chunks' LocalTracks

**Timing:** Immediately after ChunkProcessor completes new chunk

---

## The Problem: YOLO ID Discontinuity

**Issue:** YOLO assigns new tracking IDs each chunk.

**Example:**
```
Chunk N:   YOLO_ID=3 (Gardener, T=0-12s)
Chunk N+1: YOLO_ID=5 (same person, T=10-22s, YOLO reassigned ID)
```

**Solution:** Match tracks in 2-second overlap zone (T=10-12s) using spatial continuity + appearance similarity.

---

## Overlap Zone Matching

### What Is the Overlap Zone

**Last 2 seconds of chunk N** overlaps with **first 2 seconds of chunk N+1**.

For chunks processed every 10s:
- Chunk N: T=0-12s (overlap zone: T=10-12s)
- Chunk N+1: T=10-22s (overlap zone: T=10-12s)

**Why 2 seconds:**
- Long enough for reliable matching (multiple frames)
- Short enough to avoid ambiguity (agents don't move far in 2s)

---

### Matching Algorithm

**For each LocalTrack in chunk N+1:**

1. **Filter candidates from chunk N:**
   - Must be visible in overlap zone (T=10-12s)
   - Must not already be linked (next_track == None)

2. **Match by spatial continuity:**
   - Compare exit position of chunk N track with entry position of chunk N+1 track
   - Distance < threshold → candidate match

3. **If multiple candidates, use appearance similarity:**
   - Compare Re-ID embeddings (body appearance)
   - Highest similarity wins

4. **One-to-one matching:**
   - Each chunk N track links to at most one chunk N+1 track
   - Each chunk N+1 track links to at most one chunk N track

**See:** L12_TemporalMatching.md for detailed algorithm (Hungarian matching, similarity thresholds)

---

## Matching Scenarios

### Single Agent in Overlap (Deterministic)

```
Chunk N:   Track 3 (Gardener, exits at cell (4,3) in overlap zone)
Chunk N+1: Track 5 (enters at cell (4,3) in overlap zone)

Match: Deterministic (only one candidate)
  track_3.next_track = (chunk_N+1, 5)
  track_5.previous_track = (chunk_N, 3)
```

---

### Multiple Agents in Overlap (Appearance Similarity)

```
Chunk N:   Track 3 (Person A, red shirt, exits cell (3,2))
           Track 7 (Person B, blue shirt, exits cell (5,4))

Chunk N+1: Track 5 (red shirt, enters cell (3,2))
           Track 9 (blue shirt, enters cell (5,4))

Match by appearance + position:
  track_3.next_track = (chunk_N+1, 5)  # Red shirt match
  track_7.next_track = (chunk_N+1, 9)  # Blue shirt match
```

**See:** L13_ReID.md for Re-ID embedding extraction and comparison

---

### Track Terminates Before Overlap (No Link)

```
Chunk N:   Track 3 (Person leaves frame at T=7s)
           exit_reason = "left_frame"
           next_track = None  # Agent left camera

Chunk N+1: No corresponding track

Result: Track 3 stands alone (no next_track)
```

---

### Track Starts Mid-Chunk (No Previous Link)

```
Chunk N:   No track

Chunk N+1: Track 5 (Person enters frame at T=15s)
           previous_track = None  # First appearance on this camera

Result: Track 5 starts new chain
```

---

## Identity Propagation

### If Previous Track Already Has Identity

When linking, propagate global_agent_id and identity_candidates:

```python
# Chunk N track already resolved
track_N.global_agent_id = 123

# Link to chunk N+1
track_N.next_track = (chunk_N+1, 5)
track_N+1.previous_track = (chunk_N, 3)

# Propagate identity
track_N+1.global_agent_id = 123  # Inherit from previous
```

**Why:** Avoids re-resolving identity for every chunk. IdentityResolver only needs to process tracks without global_agent_id.

---

### If Previous Track Is Uncertain

```python
# Chunk N track uncertain
track_N.identity_candidates = {Agent_A, Agent_B}

# Link to chunk N+1
track_N.next_track = (chunk_N+1, 5)
track_N+1.previous_track = (chunk_N, 3)

# Propagate uncertainty
track_N+1.identity_candidates = {Agent_A, Agent_B}  # Inherit uncertainty
```

---

## Concurrent Agents

### How YOLO Handles Multiple People

YOLO assigns different local_ids to different people in the same chunk.

```
Chunk N:   Track 3 (Person A)
           Track 7 (Person B)
           concurrent_tracks = [7] on track 3
           concurrent_tracks = [3] on track 7
```

### TemporalLinker Handles Each Independently

Each person gets matched separately across chunks:

```
Chunk N:   Track 3 (Person A) → matches Track 5 (Chunk N+1)
           Track 7 (Person B) → matches Track 9 (Chunk N+1)
```

**No special handling needed.** YOLO already separated them.

**Note:** Concurrent tracks (groups) excluded from grid learning to keep data clean.

**See:** L12_ConcurrentTracking.md for details

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - TemporalLinker overview
- **L2_Strategy.md** - Why link tracks over time

### Previous Handler
- **L3_ChunkProcessing.md** - Creates LocalTracks

### Next Handler
- **L3_IdentityResolution.md** - Assigns global_agent_id (cross-camera)

### Implementation
- **L12_TemporalMatching.md** - Detailed matching algorithm (Hungarian, thresholds)
- **L13_ReID.md** - Re-ID embedding extraction and comparison
- **L13_Detection.md** - LocalTrack data structure
