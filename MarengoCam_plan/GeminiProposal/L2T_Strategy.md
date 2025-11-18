# L2T: Architectural Strategy & Principles

This document outlines the core technical strategies and design principles that govern the MarengoCam architecture.

## 1. The `LocalTrack` is the Fundamental Unit

All system state is derived from a single, canonical data structure: the `LocalTrack`. There are no separate stored objects for agents, timelines, or constraints.

```python
class LocalTrack:
    """A continuous observation of one agent on one camera in one chunk."""
    # Core Identity
    local_id: int           # YOLO's tracking ID (chunk-scoped)
    camera_id: str
    chunk_id: int

    # Observational Data
    detections: List[Detection]
    start_time: float
    end_time: float
    entry_cell: Tuple[int, int]
    exit_cell: Tuple[int, int]

    # Temporal Relationships (Same Camera)
    previous_track: Optional[Tuple[str, int, int]] # (camera_id, chunk_id, local_id)
    next_track: Optional[Tuple[str, int, int]]

    # Identity Assignment (Cross-Camera)
    global_agent_id: Optional[int]
    identity_candidates: Set[int]

    # Contextual Flags
    inside_vehicle: bool
    at_portal: Optional[str]
    exit_reason: str  # e.g., "left_frame", "chunk_boundary"
```

## 2. `GlobalAgent` is a Query, Not a Stored Object

The concept of a `GlobalAgent` is a query-time aggregation of all `LocalTrack`s that share the same `global_agent_id`. This eliminates data synchronization problems and ensures a single source of truth.

```python
def get_agent_timeline(agent_id: int) -> List[LocalTrack]:
    # Query the database for all LocalTracks with the given agent_id
    return db.query(LocalTrack).filter(global_agent_id == agent_id).order_by(start_time).all()
```

## 3. Merge Across Cameras Using a Strict Evidence Hierarchy

To ensure merges are accurate and debuggable, the `IdentityResolver` must process evidence in a specific, non-negotiable order. Stronger evidence is always processed first.

1.  **Face Recognition:** A high-confidence match (e.g., ≥ 0.75) is definitive.
    => **Result: `AUTO-MERGE`**
2.  **Grid Overlap:** The track's start/end times and locations indicate it was in a known camera overlap zone for less than a threshold (e.g., < 0.4s). Only applies if a single agent is present.
    => **Result: `AUTO-MERGE`**
3.  **Grid Transition + Alibi Check:** The track's timing and path match a learned transition in the `6x6 grid`, AND an alibi check confirms the agent was not seen elsewhere at the same time.
    => **Result: `AUTO-MERGE`**
4.  **Ambiguous Evidence:** Evidence is plausible but not definitive (e.g., multiple candidates fit the grid timing).
    => **Result: `QUEUE FOR REVIEW`** (Populate `identity_candidates`)
5.  **Contradictory Evidence:** Evidence proves two tracks are different (e.g., face mismatch, failed alibi check).
    => **Result: `REJECT`**

## 4. Accept and Persist Uncertainty

If evidence is insufficient for a high-confidence merge, the system must **not** force a decision. Instead, it must store the uncertainty.

-   **Mechanism:** The `LocalTrack.identity_candidates` field is populated with a set of potential `global_agent_id`s.
-   **Resolution:** The uncertainty persists until the `EvidenceProcessor` resolves it based on new information (e.g., a clearer face image, a human decision).
-   **Principle:** It is better to have an incomplete timeline than an incorrect one.

## 5. Learn Only from Clean, Validated Data

The system's intelligence (learned grids, face models) improves over time. To prevent cascading errors, learning models are trained **only** on unambiguous, high-confidence data.

-   **Grid Learning:** The `6x6 grid` of inter-camera travel times is updated only from `AUTO-MERGE`d tracks where a single agent was involved.
-   **Face Library:** The face recognition model is trained only on faces from tracks with a validated `global_agent_id` (either from a high-confidence merge or human review).

## 6. No Arithmetic on Evidence

The system avoids combining different types of evidence using weighted scores or probabilities (e.g., `0.5*face_confidence + 0.3*time_plausibility`).

-   **Mechanism:** Evidence is treated as a set of discrete, countable facts. A decision is made through a process of elimination based on the evidence hierarchy.
-   **Principle:** This makes the decision-making process transparent, debuggable, and less prone to "magic number" tuning.
