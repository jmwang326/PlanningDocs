# Level 3 - Temporal Linking (TemporalLinker)

## Purpose
Link LocalTracks across chunk boundaries (same camera). Handles YOLO ID discontinuity.

---

## Input/Output

**Input:**
- New chunk's LocalTracks
- Previous chunk's LocalTracks (overlap zone)

**Output:** Updated previous_track/next_track pointers

**Timing:** Immediately after ChunkProcessor

---

## The Problem: YOLO ID Discontinuity

YOLO assigns new tracking IDs each chunk.

**Example:**
```
Chunk N:   YOLO_ID=3 (person, T=0-12s)
Chunk N+1: YOLO_ID=5 (same person, T=10-22s, YOLO reassigned ID)
```

**Solution:** Match tracks in 2s overlap zone (T=10-12s) using spatial continuity + appearance similarity.

---

## Overlap Zone Matching

**For each LocalTrack in chunk N+1:**

1. **Filter candidates from chunk N:**
   - Visible in overlap zone (T=10-12s)
   - Not already linked (next_track == None)

2. **Match by spatial continuity:**
   - Exit position of chunk N ≈ entry position of chunk N+1
   - Distance < threshold → candidate match

3. **If multiple candidates, use appearance similarity:**
   - Compare Re-ID embeddings
   - Highest similarity wins

4. **One-to-one matching:** Each track links to at most one track

---

## Identity Propagation

**If previous track has identity:**
```python
track_N.global_agent_id = 123
track_N.next_track = (chunk_N+1, 5)
track_N+1.previous_track = (chunk_N, 3)
track_N+1.global_agent_id = 123  # Inherit
```

**If previous track uncertain:**
```python
track_N.identity_candidates = {Agent_A, Agent_B}
track_N+1.identity_candidates = {Agent_A, Agent_B}  # Inherit uncertainty
```

---

## Concurrent Agents

YOLO assigns different local_ids to different people in same chunk:

```
Chunk N:   Track 3 (Person A) → matches Track 5 (Chunk N+1)
           Track 7 (Person B) → matches Track 9 (Chunk N+1)
```

**No special handling needed.** YOLO already separated them.

**Note:** Concurrent tracks (groups) excluded from grid learning (keep data clean).
