# Level 13 - Track Merging Engine

## 1. Purpose
This document provides the technical specification for the Track Merging Engine. The primary responsibility of this engine is to identify and link track segments from different cameras that belong to the same agent, using the `6x6` inter-camera adjacency grid as its core mechanism.

---

## 2. Core Component: `6x6` Inter-Camera Adjacency Grid

### 2.1. Data Structure
- **Matrix:** A `6x6` matrix of 32-bit integers is maintained for each **directed camera pair** (e.g., `FrontDoor` -> `Driveway` is distinct from `Driveway` -> `FrontDoor`).
- **Agent Classes:** Separate sets of matrices are maintained for each major agent class (e.g., `person`, `vehicle`).
- **Cell Value:** Each cell `(row, col)` in the matrix stores the **minimum observed travel time in milliseconds** between the source camera's exit cell (`row`) and the destination camera's entry cell (`col`).

### 2.2. Special Cell Values
- **`0` (Zero):** Indicates a **confirmed camera overlap zone**. A value of `0` means an agent can be in both cells simultaneously.
- **`-1` (or `MAX_INT`):** Represents an **unlearned path**. No high-confidence merge has yet been observed between these two cells.

### 2.3. Grid Initialization
- All cells in all grid matrices are initialized to `-1` to signify that no paths have been learned.

---

## 3. Algorithm: Merge Candidate Selection

This algorithm is triggered when an agent's track ends on a source camera.

1.  **Input:**
    - `source_track`: The track that just ended.
    - `source_camera_id`: The camera where the track ended.
    - `exit_cell_6x6`: The `6x6` grid cell where the agent was last seen.
    - `exit_time`: The timestamp when the agent was last seen.

2.  **Process:**
    a. **Query for New Tracks:** The engine queries the `Track State Manager` for a list of new tracks that have appeared on other cameras within a **maximum plausible time window** (a configurable system parameter, e.g., 30 seconds).

    b. **Filter Candidates:** For each `candidate_track` in the list:
        i. Get the `dest_camera_id`, `entry_cell_6x6`, and `entry_time`.
        ii. Look up the corresponding grid: `grid = get_grid(source_camera_id, dest_camera_id, agent_class)`.
        iii. Retrieve the learned minimum time: `min_time_ms = grid[exit_cell_6x6][entry_cell_6x6]`.
        iv. **If `min_time_ms` is `-1` (unlearned):** The candidate is considered low-confidence and is likely rejected unless other strong evidence (like a face match) exists.
        v. **Calculate Time Delta:** `delta_time_ms = entry_time - exit_time`.
        vi. **Check Plausibility:** The candidate is considered plausible if its `delta_time_ms` is within a configurable tolerance of the learned minimum. For example: `delta_time_ms < (min_time_ms * [tolerance_factor]) + [grace_period_ms]`.
            - The `tolerance_factor` (e.g., `2.5`) allows for agents moving slower than the fastest observed time.
            - The `grace_period_ms` (e.g., `2000ms`) adds a small fixed buffer.

3.  **Output:**
    - **One Plausible Candidate:** The system has a high-confidence merge candidate. This can be routed for an automatic merge, subject to confirmation by the Re-ID engine.
    - **Multiple Plausible Candidates:** The decision is ambiguous. The list of candidates is passed to the `Evidence Engine` to gather more information (Re-ID scores, appearance data) for a final decision by the `Inference Manager` (which may involve LLM/human review).
    - **No Plausible Candidates:** No merge is made. The new track becomes a new `Unknown` agent.

---

## 4. Algorithm: Grid Learning and Updating

The grid is continuously refined as new, high-confidence merges are validated. This process is strictly filtered to prevent data pollution.

1.  **Trigger:** A merge is confirmed with **high confidence** (e.g., via face match or human validation).

2.  **Learning Filter:** Before updating the grid, the merge context is checked against a strict filter. The update proceeds **only if all conditions are met**:
    - `source_track.is_solo_track == true`
    - `dest_track.is_solo_track == true`
    - The merge was not the result of an ambiguous decision that required LLM or human review for a tie-break.

    A "solo track" is one where no other agents were concurrently detected within the frame, ensuring the observation represents an individual's behavior, not a group's.

3.  **Input:**
    - The validated `source_track` and `dest_track` that passed the filter.
    - The `delta_time_ms` for the merge.

3.  **Process:**
    a. Retrieve the relevant `grid`, `exit_cell_6x6`, and `entry_cell_6x6`.
    b. Read the current value from the grid: `current_min_time_ms = grid[exit_cell_6x6][entry_cell_6x6]`.
    c. **Update Logic:**
        i. **If `current_min_time_ms` is `-1` (unlearned):** The new `delta_time_ms` becomes the first learned value.
        ii. **If `delta_time_ms` < `current_min_time_ms`:** A new fastest path has been found. The grid is updated with the new, lower `delta_time_ms`.
        iii. Otherwise, the grid is not changed.

4.  **Overlap Learning:**
    - If a merge is validated for an agent that is visible on both cameras *at the same time*, the `delta_time_ms` will be `~0`.
    - When this occurs, the corresponding grid cell is updated to `0`, permanently marking it as a direct overlap zone.

---

## 5. Algorithm: Process of Elimination

**Status: Needs Refinement**

This logic is applied as a secondary filter when the initial Merge Candidate Selection results in multiple plausible candidates. It serves to reduce ambiguity by using an agent's known location to rule out impossible merges.

1.  **Trigger:** The Merge Candidate Selection algorithm produces multiple plausible candidates for a single `source_track`.
2.  **Input:** A list of `plausible_candidate_tracks`.
3.  **Process:**
    a. For each `candidate_track` in the list, identify its associated `candidate_agent`.
    b. Query the `Track State Manager` to determine if this `candidate_agent` has any other active tracks on *any other camera* that overlap in time with the `source_track`.
    c. **Elimination:** If the `candidate_agent` was verifiably present on a different, non-adjacent camera at the same time, it is logically impossible for it to also be the agent in the `source_track`. This `candidate_track` is removed from the list of possibilities.
4.  **Output:**
    - A reduced list of candidate tracks. If the list is reduced to a single candidate, it can be treated as a high-confidence match and routed for Re-ID confirmation. If multiple candidates still remain, the ambiguity is passed to the `Inference Manager` for a more detailed review.
