# L3T: Identity Resolution Technical Details

## Component: `IdentityResolver`

This document provides the specific technical parameters and logic for the `IdentityResolver` component.

---

## Evidence Hierarchy Implementation

The hierarchy must be processed in this exact order.

### 1. Face Recognition Match
- **Trigger:** A `LocalTrack` contains a face that can be compared against the known face library.
- **Condition:** `face_match_confidence >= 0.75`
- **Action:** `AUTO-MERGE`. Assign `track.global_agent_id = matched_agent_id`.

### 2. Grid Overlap Match
- **Trigger:** A track begins, and a candidate track from another camera ended recently.
- **Condition:** `(track_B.start_time - track_A.end_time) < 0.4s` AND `max_concurrent_agents_in_overlap_zone == 1`.
- **Meaning:** The time gap is so small it implies a shared physical space (camera overlap). The single agent condition prevents ambiguity.
- **Action:** `AUTO-MERGE`. Assign `track_B.global_agent_id = track_A.global_agent_id`.

### 3. Grid Transition Match + Alibi Check
- **Trigger:** A track begins, and a candidate track from another camera ended at a time consistent with a learned path.
- **Condition:**
    1. Calculate `travel_time = track_B.start_time - track_A.end_time`.
    2. Get `grid_time` from the `6x6 grid` for the corresponding camera pair and cells.
    3. Check if `abs(travel_time - grid_time) < variance_threshold`.
    4. If timing is plausible, perform an **Alibi Check**.
- **Alibi Check:** Determines if any *other* active agent could also have plausibly made it to `track_B`'s entry point in time.
    - **If `alibi_check_passes()` (no other agent could have done it):**
        - **Action:** `AUTO-MERGE`. By process of elimination, `track_B` must belong to `track_A`'s agent.
    - **If `alibi_check_fails()` (another agent is also a plausible source):**
        - **Action:** `QUEUE FOR REVIEW`. Assign `track_B.identity_candidates = {agent_A, other_plausible_agents...}`.

### 4. No Plausible Match
- **Trigger:** The track cannot be matched with any existing agent via the methods above.
- **Action:** Create a new agent. Assign `track.global_agent_id = create_new_agent()`.

---

## Alibi Check (Process of Elimination)

**Purpose:** Rule out candidates who were seen elsewhere at incompatible times.

**Algorithm:**
1.  **Find Candidates:** `candidates = all_agents_seen_recently(entry_track.start_time - 30s)`
2.  **Check Travel Time:**
    -   For each candidate, calculate `travel_time = entry_track.start_time - candidate.end_time`.
    -   Compare against `expected_time = grid[(candidate.camera, entry_track.camera)]`.
    -   If `travel_time < expected_time * 0.5`, the candidate is **IMPOSSIBLE** (too fast).
    -   Otherwise, the candidate is **PLAUSIBLE**.
3.  **Decision:**
    -   **0 Plausible:** `DEFINITIVE` (Must be from current candidate/new agent).
    -   **1 Plausible:** `AMBIGUOUS` (Binary choice).
    -   **2+ Plausible:** `AMBIGUOUS` (Multi-way choice).

**Cascading Elimination:**
When an agent is confirmed in one track, remove that agent from the `identity_candidates` of all other contemporaneous uncertain tracks. If a track's candidate set is reduced to one, auto-resolve it and recurse.

---

## Grid Learning (6x6 Spatial Model)

**Purpose:** Learn minimum travel times between cameras to support the Alibi Check and Grid Transition Match.

**Concept:**
-   Overlay a 6x6 grid on each camera's FOV.
-   Learn `min_travel_time` for every path: `(Cam_A, Exit_Cell) -> (Cam_B, Entry_Cell)`.

**Learning Process:**
1.  **Trigger:** A validated merge (Face Match ≥ 0.75 OR Human Approval).
2.  **Clean Data Check:**
    -   `max_concurrent_agents == 1` (Solo track).
    -   `duration >= 3s` (Sustained movement).
    -   Not a portal transition.
3.  **Update:**
    -   `travel_time = track_B.start_time - track_A.end_time`
    -   `if travel_time < current_grid_min: update_grid(...)`

**Bootstrap Phases:**
-   **Phase 1 (Days 1-3):** No grid data. Rely on face match/human review.
-   **Phase 2 (Days 4-14):** Partial grid. Use where available.
-   **Phase 3 (Days 15+):** Grid complete. Target > 80% auto-merge.

---

## Portal Transitions

**Purpose:** Handle identity merges where agents disappear for unpredictable durations (e.g., entering a garage).

**Logic:**
-   **Constraint:** Remove the *maximum* time constraint, but strictly enforce the *minimum* travel time (grid logic).
-   **Evidence:** Prioritize Face Recognition and Appearance Similarity (Re-ID) over temporal proximity.
-   **Single-Entry Inference:** If only one person enters a portal and one person exits later, infer a match even with weak visual evidence.

---
