# 02 - Component: Identity Resolver (The Brain)

## Purpose
The `IdentityResolver` is the core logic engine that converts fragmented `LocalTracks` (from individual cameras) into coherent `GlobalEntities` (unique people/vehicles).

**Philosophy:**
- **Conservative:** Better to have two "Strangers" than one "Frankenstein" (wrongly merged).
- **Evidence-Based:** Merges require specific evidence (Face, Spatial, or Temporal).
- **Direct-Link Only:** We do not infer multi-hop paths. If A->B is not in the Grid, we don't assume it's possible.

---

## 1. Interface Contract

### 1.1. Input
The `ChunkProcessor` calls this after finishing a track.
```python
def resolve_identity(new_track: LocalTrack) -> IdentityResult:
    """
    Input: A finished LocalTrack with start/end times, zones, and optional embeddings.
    Output: The GlobalEntityID assigned to this track.
    """
```

### 1.2. Output
Updates the `local_tracks` table:
- Sets `global_entity_id`.
- Sets `merge_reason` (e.g., "face_match", "spatial_link").
- Sets `confidence` (0.0 to 1.0).

---

## 2. Resolution Logic (The Pipeline)

The resolver runs a series of checks in order of precedence.

### Step 1: The "Face Trump Card" (High Confidence)
If the track has a high-quality face embedding.
1.  Query `global_entities` for face match (Cosine Similarity > 0.75).
2.  **If Match:**
    - Assign `global_entity_id`.
    - Update `last_seen_at`.
    - **Return (AUTO-MERGE).**

### Step 2: Temporal Linking (Same Camera)
Did this person just stand still or get occluded?
1.  Query `local_tracks` on **Same Camera**.
2.  Filter: `end_time` within `[new_start - 2s, new_start]`.
3.  Check: **IoU (Intersection over Union)** of bounding boxes > 0.3.
4.  **If Match:**
    - Inherit `global_entity_id` from the previous track.
    - **Return (AUTO-MERGE).**

### Step 3: Spatial Overlap (Overlapping Cameras)
Are two cameras looking at the same physical spot?
1.  Query `local_tracks` on **Neighbor Cameras** (defined in Config as "Overlapping").
2.  Filter: Time overlap (tracks exist simultaneously).
3.  Check: **Grid Cell Alignment** (e.g., CamA Zone 3 overlaps CamB Zone 1).
4.  **If Match:**
    - Merge IDs.
    - **Return (AUTO-MERGE).**

### Step 4: Spatial Transition (The Grid)
Did the person walk from Camera A to Camera B?
1.  Query `local_tracks` on **All Cameras**.
2.  Filter: `end_time` is before `new_start`.
3.  Loop through candidates:
    - Check `grid_links` table for row `(candidate.cam, candidate.exit, new.cam, new.entry)`.
    - **Constraint:** If no row exists, **SKIP** (Direct Link Only).
    - Get `avg_time` and `std_dev`.
    - Calculate `travel_time = new.start - candidate.end`.
    - Score: `Z-Score = abs(travel_time - avg_time) / std_dev`.
4.  **Decision:**
    - If `Z-Score < 2.0` (within 2 sigma): **Candidate for Merge**.
    - If multiple candidates: **Ambiguous -> Create New Identity**.
    - If single strong candidate: **Return (AUTO-MERGE).**

### Step 5: Default
No matches found.
1.  Create new `GlobalEntity`.
2.  **Return (NEW IDENTITY).**

---

## 3. The Learning Loop (Feedback)

How the system gets smarter.

### 3.1. Implicit Learning (Auto-Updates)
When **Step 1 (Face)** or **Step 2 (Temporal)** triggers a merge between two different cameras:
1.  Calculate `travel_time`.
2.  Update `grid_links` stats (Running Average/StdDev).
    - *Note:* We use a "Welford's Algorithm" for online variance updates to avoid storing all history.

### 3.2. Explicit Learning (Human Feedback)
Triggered via API `POST /decisions/merge`.
1.  User says "Track A is Track B".
2.  System performs the merge in DB.
3.  System calculates `travel_time`.
4.  System updates `grid_links` with a **higher weight** (e.g., counts as 5 samples) because it is human-verified.

---

## 4. Resilience & Edge Cases

### 4.1. The "Doppelgänger" Problem
- **Scenario:** Two people walk side-by-side.
- **Defense:** If `IdentityResolver` finds a candidate that is *already active* (overlapping time) on another camera, it **rejects** the merge (unless "Spatial Overlap" logic applies).
- **Rule:** "An entity cannot be in two non-overlapping places at once."

### 4.2. Cold Start (Empty Grid)
- Initially, `grid_links` is empty.
- **Behavior:** Step 4 (Spatial Transition) never triggers.
- **Result:** System creates many "New Identities."
- **Fix:** Human uses Dashboard to merge them -> `grid_links` populates -> System starts auto-merging.

### 4.3. Grid Corruption
- **Scenario:** Bad merge creates a "1 second" link for a "10 second" path.
- **Defense:** Outlier rejection in the learning algorithm. If a new sample is > 5 sigma away from current mean, ignore it (or flag for review) instead of ruining the average.
