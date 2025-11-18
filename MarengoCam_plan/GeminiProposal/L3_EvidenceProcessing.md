# L3: Evidence Processing

## Component: `EvidenceProcessor`

## Purpose

The `EvidenceProcessor` is an asynchronous, event-driven component responsible for resolving identity uncertainty and correcting merge errors. It acts as the central hub for incorporating new information into the system after the initial `IdentityResolver` pass.

While the `IdentityResolver` makes the initial attempt at merging, the `EvidenceProcessor` handles the long-term lifecycle of a track's identity, reacting to new intelligence as it becomes available.

## High-Level Process

The component is triggered when new evidence arrives that pertains to a `LocalTrack`. This evidence can take several forms:

1.  **Automated Evidence:** A new, higher-quality face image becomes available, leading to a high-confidence match from the face recognition service.

2.  **Adjudication Evidence:** An ambiguous case from the "Review Queue" is submitted to an LLM or a human for a decision. The resulting decision is fed back as evidence.

3.  **Corrective Evidence:** A human operator identifies an incorrect merge in the timeline and initiates an "unmerge" or "split" action. This is treated as definitive evidence that two tracks belong to different agents.

Upon receiving evidence, the `EvidenceProcessor` applies the new information, resolves the `global_agent_id` for the track in question, and then triggers a cascade of secondary updates to ensure the system's state remains consistent.

## Key Responsibilities

-   **Resolving Uncertainty:** Applying new evidence to `LocalTrack`s that have a set of `identity_candidates`.
-   **Correcting Errors:** Handling "unmerge" events to split incorrectly merged tracks.
-   **Propagating Knowledge:** Ensuring that once a track's identity is resolved, that resolution is propagated forward, backward, and sideways to all related tracks.
-   **Triggering System Learning:** Initiating updates to the `6x6 Grid` and face libraries after a merge has been validated by high-confidence evidence.
-   **Managing the Review Queue:** Prioritizing and preparing ambiguous cases for review by an LLM or human.

## Output

The `EvidenceProcessor` does not have a direct output in the pipeline. Instead, it modifies the state of `LocalTrack` objects in the database, resolving uncertainty and ensuring the global timeline remains accurate and consistent.
