# L3T: Evidence Processing Technical Details

## Component: `EvidenceProcessor`

This document provides the specific technical parameters and logic for the `EvidenceProcessor` component.

---

## Evidence Handling

The processor is designed to handle three main categories of event-driven evidence.

### 1. Automated Evidence (e.g., Face Recognition)
- **Source:** Asynchronous AI services.
- **Logic:**
    - If `confidence >= 0.75`, the evidence is considered definitive. The track's `global_agent_id` is resolved.
    - If `confidence < 0.75`, the evidence is weak. The potential `agent_id` is added to the `identity_candidates` set but does not trigger a resolution.

### 2. Adjudication Evidence (LLM or Human Review)
- **Source:** The Review Queue GUI or an API call from an LLM service.
- **Logic:**
    - **Human decisions are considered ground truth and are always trusted.**
    - High-confidence LLM decisions (`decision: "merge"`, `confidence: "high"`) are treated as definitive.
    - A `reject` decision removes the specified `agent_id` from the `identity_candidates` set.

### 3. Corrective Evidence (Human-driven Unmerge/Split)
- **Source:** A user action from the timeline review GUI.
- **Logic:**
    1. The user selects a `LocalTrack` that was incorrectly merged.
    2. The system performs a "split":
        a. A new `global_agent_id` is created.
        b. The selected `LocalTrack` and all subsequent tracks in its chain (linked via `next_track` and `merged_tracks`) are reassigned to this new `global_agent_id`.
    3. This action is logged as a definitive "reject" event between the original agent and the newly created one.

---

## Resolution and Propagation Cascade

Once a piece of evidence resolves a track's identity, a cascade is triggered to ensure system-wide consistency.

1.  **Apply Resolution:**
    -   `track.global_agent_id = resolved_agent_id`
    -   `track.identity_candidates.clear()`

2.  **Propagate to Linked Tracks:**
    -   Recursively traverse the track's relationships (`next_track`, `previous_track`, `merged_tracks`).
    -   Apply the same `resolved_agent_id` to all connected tracks that are still uncertain.

3.  **Apply Process of Elimination:**
    -   For the given `resolved_agent_id`, find all other contemporaneous `LocalTrack`s elsewhere in the system that list it as a candidate.
    -   Remove the `resolved_agent_id` from their `identity_candidates` sets.
    -   If any of a track's `identity_candidates` sets are reduced to a single entry, that track is now also resolved. This can trigger a new propagation cascade.

---

## System Learning on Confirmation

When a merge is validated by high-confidence evidence (e.g., human review, face match ≥ 0.75), it triggers updates to the system's knowledge base.

-   **Face Library Update:**
    -   The face crops from the newly confirmed `LocalTrack` are added to the training library for the confirmed `global_agent_id`. This refines the agent's face model.

-   **`6x6` Grid Update:**
    -   If the merge was between two `LocalTrack`s on different cameras and represents a clean, single-agent path, the travel time is submitted to the grid learning module.
    -   The module updates the minimum travel time for that specific inter-camera cell-to-cell path.

---

## Review Queue Management

**Purpose:** Prioritize ambiguous cases for human/LLM review.

**Prioritization Logic:**
-   **Score:** `priority = (w1 * recency) + (w2 * simplicity) + (w3 * impact)`
    -   **Recency:** Fresh evidence is more valuable.
    -   **Simplicity:** Binary choices (A vs B) are quick wins.
    -   **Impact:** Resolving a track that blocks many others (Process of Elimination) is high value.

**Dynamic Pruning:**
-   Automatically remove items if new evidence (e.g., delayed face match) resolves the uncertainty.

**Hybrid Strategy:**
-   **LLM:** Assigned simple, binary visual comparisons.
-   **Human:** Assigned complex, multi-candidate, or high-impact cases.
