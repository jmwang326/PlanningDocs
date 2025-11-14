# Level 3 - Track Merging Engine - Tactical

## 1. Purpose

The **Track Merging Engine** is the primary decision-making component for agent identity. Its purpose is to act as a pure, simple, and highly readable rules engine. It receives a factual `EvidenceReport` from the `Evidence Engine` and, based on a clear hierarchy of rules, makes the final decision on whether to merge two tracks, reject the merge, or escalate for review.

## 2. Key Responsibilities

- **Decision Logic:** The engine's sole responsibility is to execute a set of predefined rules against an `EvidenceReport`. It does not gather data; it only interprets the data it is given.

- **Interpreting Grid Data:** A primary responsibility is to interpret the `6x6` inter-camera grid data provided in the `EvidenceReport`. It uses this to determine if a transition between two cameras is spatially and temporally plausible according to learned historical data.

- **Applying Process of Elimination:** When multiple plausible candidates are suggested by the grid, the engine uses the known locations of other agents to rule out impossible merges.

- **Rule Hierarchy Implementation:** It will strictly implement the evidence hierarchy defined in `L2_Strategy.md`. The logic will follow a clear, cascading flow:
    1.  **Check for Definitive Evidence:** Does the report contain a high-confidence face match? If yes, `AUTO_MERGE`.
    2.  **Check for Plausible Grid Transition:** If no face match, does the `EvidenceReport` indicate a plausible transition according to the learned timings in the `6x6` grid?
    3.  **Combine with Secondary Evidence:** If the grid transition is plausible, combine it with other evidence like Re-ID scores to decide between `AUTO_MERGE` (if Re-ID is also high-confidence) or `REVIEW_LLM` (if Re-ID is ambiguous).
    4.  **Check for Conflicting Evidence:** Does the evidence conflict (e.g., a plausible grid transition but a clear face mismatch)? If yes, `REVIEW_HUMAN`.
    5.  **Default to Rejection:** If there is no plausible grid transition and no other strong evidence, the default decision is `REJECT`.

- **Issuing Commands:** Based on its decision, the engine will issue a command back to the `Track State Manager`. This could be:
    - `merge_tracks(source_track, dest_track)`
    - `add_to_review_queue(source_track, dest_track, reason)`
    - `log_rejected_merge(source_track, dest_track, reason)`

### 2.1. Handling Portals
A **Portal** is a special type of grid cell representing a location where an agent can become concealed for an indeterminate amount of time (e.g., a garage, a building entrance). Standard grid-based travel time logic does not apply.

- **Exit/Entry Plausibility:** When a track disappears into a Portal and a new track later emerges from the *same Portal*, the transition is considered plausible regardless of the time gap.
- **Reliance on Secondary Evidence:** Since time is not a reliable factor, the decision to merge will depend more heavily on secondary evidence, such as appearance similarity (Re-ID) and the process of elimination. A high Re-ID score for a track emerging from the same portal would lead to an `AUTO_MERGE`, whereas a lower score might require `REVIEW_LLM`.

## 3. Operational Flow

1.  **Invocation:** The `Track State Manager` asks the `Track Merging Engine` to evaluate a potential merge.
2.  **Delegation:** The `Merging Engine` immediately requests an `EvidenceReport` from the `Evidence Engine`.
3.  **Rule Execution:** Upon receiving the report, the `Merging Engine` executes its internal rule set, with the `6x6` grid plausibility check being a key step. The code will be structured as a series of simple, readable `if/elif/else` statements that directly mirror the evidence hierarchy.
4.  **Decision & Command:** The engine reaches a decision (`AUTO_MERGE`, `REVIEW_LLM`, `REJECT`, etc.) and translates this into a specific command.
5.  **Response:** The command is returned to the `Track State Manager` for execution. The `Merging Engine`'s job is now complete.

## 4. Strategic Importance

This component is the heart of the system's "brain." By keeping it pure—strictly a rules engine—it becomes incredibly transparent and auditable. When a strange merge occurs, we can look directly at the `EvidenceReport` it received (including the `6x6` grid data) and the rule it executed to understand *why* it made that decision. This separation prevents the logic from becoming a tangled mess of data gathering and decision-making, which is critical for long-term maintenance and debugging. It is a classic example of separating "policy" (the rules in the Merging Engine) from "mechanism" (the data gathering in the Evidence Engine).
