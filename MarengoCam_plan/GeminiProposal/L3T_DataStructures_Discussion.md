# L3T: Data Structures & Contracts Discussion

## 1. The Grid Metric Structure

**The Requirement:** We need to store the learned travel times between every possible location on the property to every other location.

**The Concept:** A "6x6 Grid" overlays each camera view.
-   **Source:** Camera A, Cell (x1, y1)
-   **Destination:** Camera B, Cell (x2, y2)
-   **Metric:** Minimum observed travel time.

### Proposed Structure: `GridLink`

Instead of a single massive 4D array, we store sparse links. We only care about paths that actually exist.

```python
class GridLink(BaseModel):
    # The Path
    from_camera: str
    from_cell_x: int # 0-5
    from_cell_y: int # 0-5
    
    to_camera: str
    to_cell_x: int   # 0-5
    to_cell_y: int   # 0-5
    
    # The Knowledge
    min_travel_time: float      # The fastest anyone has ever made this trip
    avg_travel_time: float      # Moving average (for anomaly detection)
    sample_count: int           # Confidence metric (n=1 is weak, n=100 is strong)
    last_updated: datetime      # For decaying old data
```

### The "Summary Metric" (User Request)

The user suggested: *“store a mintime for each grid for each pair which gets updated every time the 6x6x6x6 changes”*

This is a **Camera-to-Camera Summary**. It allows for a fast "pre-filter" before doing the expensive cell-level lookup.

```python
class CameraPairSummary(BaseModel):
    from_camera: str
    to_camera: str
    
    # The absolute fastest time between ANY two points on these cameras
    global_min_time: float 
```

### Pros & Cons

**Pros:**
1.  **Performance:** The `CameraPairSummary` acts as a Bloom filter. If `(Track_B.start - Track_A.end) < global_min_time`, we can instantly reject the match without querying the detailed grid.
2.  **Sparcity:** Storing `GridLink` rows in a DB table (rather than a dense tensor) saves space. Most cell-to-cell combinations are physically impossible (walls, teleportation) and will never exist.
3.  **Granularity:** 6x6 is fine enough to distinguish "Door" from "Driveway" but coarse enough to gather meaningful statistics.

**Cons:**
1.  **Cold Start:** Until `sample_count > N`, the data is noisy. We need a "Bootstrap" flag.
2.  **Drift:** Physical paths change (e.g., a gate is locked). We need a decay factor or a rolling window to forget ancient history.

**Alternative:**
-   **Dense Tensor:** A fixed `[Num_Cams, 6, 6, Num_Cams, 6, 6]` float array.
    -   *Pro:* O(1) lookup, easy to load into GPU/Numpy.
    -   *Con:* Massive memory waste (99% zeros). Harder to update incrementally in a DB.

**Decision:** Use the **Sparse DB Table (`GridLink`)** + **Cached Summary (`CameraPairSummary`)**.

---

## 2. The Evidence Object

**The Requirement:** A standardized payload that represents *any* new information entering the system that might change an identity.

### Proposed Structure: `Evidence`

```python
class Evidence(BaseModel):
    # Who is this about?
    target_track_id: int
    
    # What is the source?
    source_type: str  # "face_ai", "human_review", "llm_adjudicator", "alibi_check"
    source_id: str    # e.g., "user_jmwang", "gpt-4o", "codeproject_v2"
    
    # What is the claim?
    suggested_agent_id: Optional[int] # If None, implies "New Agent" or "Reject"
    decision: str                     # "merge", "reject", "uncertain"
    
    # How strong is it?
    confidence: float                 # 0.0 to 1.0
    
    # Why? (For debugging/audit)
    reason: str                       # e.g., "Face match 0.88", "Impossible travel time"
    timestamp: datetime
```

### Pros & Cons

**Pros:**
1.  **Uniformity:** The `EvidenceProcessor` doesn't care if the info came from a human or a script. It processes `Evidence` objects generically.
2.  **Audit Trail:** We can log every `Evidence` object to a table. This creates a perfect history of *why* the system believes Agent X is Agent X.
3.  **Replayability:** We can replay the `Evidence` log to rebuild the state if the DB is corrupted.

**Cons:**
1.  **Verbosity:** Storing a record for every single face match might bloat the DB.
    -   *Mitigation:* Only store "decisive" evidence or evidence that changes state.

**Alternative:**
-   **Direct State Mutation:** Just call `track.set_identity()`.
    -   *Pro:* Simple.
    -   *Con:* No history. If the system makes a mistake, we can't trace *why*. "Chicken and Egg" bugs become impossible to debug.

**Decision:** The `Evidence` object is mandatory for a forensic system.

---

## 3. The `LocalTrack` Object (Refined)

**The Requirement:** The atomic unit of observation.

### Proposed Structure

```python
class LocalTrack(BaseModel):
    # Identity
    id: int                     # DB Primary Key
    camera_id: str
    chunk_id: int
    local_id: int               # YOLO ID
    
    # The "Truth" (Mutable)
    global_agent_id: Optional[int]
    identity_candidates: Set[int] # The "Maybe" list
    
    # Spatio-Temporal Data
    start_time: float
    end_time: float
    entry_cell: Tuple[int, int] # (x, y)
    exit_cell: Tuple[int, int]  # (x, y)
    
    # Visual Data (Lazy Loaded)
    # We do NOT store images here. We store paths or references.
    best_face_image_path: Optional[str]
    best_body_crop_path: Optional[str]
    
    # Quality Metrics
    is_stationary: bool         # Was this person standing still? (Good for face, bad for grid)
    has_face: bool
```

### Pros & Cons

**Pros:**
1.  **Lightweight:** No image blobs. Fast to query.
2.  **Explicit Uncertainty:** `identity_candidates` is a first-class citizen.

**Cons:**
1.  **Complexity:** Managing the `identity_candidates` set requires careful logic (set intersection, union) during merges.

**Decision:** This structure supports the "Process of Elimination" strategy perfectly.

---

## 4. Multi-Hop Grid Inference (A -> B -> C)

**The Question:** Should we infer the travel time for `A -> C` by summing `A -> B` and `B -> C`?

**The Logic:**
-   We know `MinTime(A, B) = 10s`.
-   We know `MinTime(B, C) = 10s`.
-   Can we assume `MinTime(A, C) >= 20s`?

**The Verdict: NO.**

**Reasoning:**
1.  **The "Shortcut" Problem:** This logic assumes that the *only* way to get from A to C is through B. In reality, there might be a direct path (e.g., a window or a door) that takes 2 seconds.
2.  **False Rejection:** If we use the inferred 20s as a lower bound, we would falsely reject a valid 2s transition as "Impossible/Teleportation".
3.  **Triangle Inequality:** Mathematically, `DirectPath <= Sum(IndirectPaths)`. The multi-hop sum gives us an *Upper Bound* on the shortest path, but the Alibi Check requires a *Lower Bound* (fastest possible time).

**Conclusion:** We must learn direct links only. If `A -> C` is never observed, the system defaults to "Unknown/Possible", which is safer than "Impossible".
