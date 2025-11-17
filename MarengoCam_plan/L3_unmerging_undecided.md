# L3 - Track Unmerging and Identity Confirmation (Undecided)

This document captures the discussion around correcting tracking errors, specifically incorrect merges. The decisions here are not final and are recorded for future review before implementation.

## Core Problem

The tracking system will inevitably make mistakes, merging two distinct `globalTrack`s (e.g., two different people) into one. These decisions cannot be permanent and irreversible. We need a mechanism for error correction.

## 1. Manual Unmerging (Track Splitting)

This is a reactive approach driven by a human operator.

-   **Trigger:** A user identifies an incorrect merge in the GUI's "Review Panel".
-   **Interface:** The user would select a `globalTrack` and the specific `localTrack` segment where the incorrect merge occurred. They would then initiate a "split" or "unmerge" command.
-   **System Action:**
    1.  A new `globalTrack` is created.
    2.  The incorrectly merged `localTrack` segment, and all subsequent segments that were part of that chain, are moved from the original `globalTrack` to the newly created one.
    3.  The `decisionLog` for both `globalTrack`s is updated to reflect this split.
-   **Dependency:** This entire process relies on maintaining a clean, ordered `decisionLog` on each `globalTrack` that records every merge event. This log is the audit trail that makes a clean split possible.

## 2. Proactive ID Confirmation

This is a preventative measure to reduce the frequency of incorrect merges for known individuals.

-   **Trigger:** A `localTrack` is a candidate for merging into a `globalTrack` that is already associated with a high-confidence known identity (e.g., "Alice" from facial recognition).
-   **System Action:**
    1.  Before committing the merge, the system performs a targeted, high-confidence identity check (e.g., comparing facial feature embeddings).
    2.  If the new evidence (the `localTrack`) strongly contradicts the established identity of the `globalTrack`, the merge is automatically rejected.
    3.  The system can then either create a new `globalTrack` for the `localTrack` or flag it for manual review.
-   **Benefit:** This acts as a quality control gate, preventing obvious errors and reducing the burden on manual reviewers.

## Open Questions

-   What is the exact data structure of the `decisionLog`?
-   How does a split affect downstream processes like `TimelineReconstruction`?
-   What is the confidence threshold for the proactive ID confirmation?

## 3. Merge Approval and System Learning

When a merge is deemed correct (either by automated high-confidence checks or manual user approval), it serves as a positive confirmation that reinforces the system's understanding. This is a critical learning event.

-   **System Action on Approval:**
    1.  The evidence from the merged `localTrack` (e.g., facial embeddings, appearance vectors) is integrated into the target `globalTrack`. This refines the composite identity of the `globalTrack`, making future matches more accurate.
    2.  The `decisionLog` for the `globalTrack` is updated to permanently record the merge event.

-   **Grid Update:**
    -   This newly enriched `globalTrack` data is the source material used to update the system's knowledge base, including the 6x6 grid. The grid is updated to reflect the new, more complete understanding of the identity, including any new faces or associations gained from the merge. This ensures the system's "memory" is current.
