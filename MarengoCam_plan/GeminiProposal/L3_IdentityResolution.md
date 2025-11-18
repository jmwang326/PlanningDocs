# L3: Identity Resolution

## Component: `IdentityResolver`

## Purpose

The `IdentityResolver` is the third and most complex component in the processing pipeline. Its mission is to solve the core multi-camera assignment problem: determining if a `LocalTrack` from one camera and a `LocalTrack` from another camera belong to the same `GlobalAgent`.

This component is where the system moves from processing single-camera data to building a unified, cross-camera understanding of an agent's activity.

## High-Level Process

The `IdentityResolver` operates on a batch of `LocalTrack`s that have been processed by the `TemporalLinker`. For each track that does not yet have a `global_agent_id`, it attempts to find a match by stepping through a strict evidence hierarchy.

### The Evidence Hierarchy

To ensure decisions are accurate and debuggable, evidence is evaluated in a specific order, from most definitive to least definitive. The component stops at the first level that provides a clear decision.

1.  **Definitive Evidence (e.g., Face Recognition):** The strongest possible evidence is checked first. If a high-confidence face match exists, the track is merged automatically.

2.  **Spatio-Temporal Logic (Grid System):** If no definitive evidence is found, the system uses the `6x6 Inter-Camera Grid` to check if the track's timing and location are plausible. This includes checking for camera overlap and learned travel times. An "alibi check" ensures no other agent could account for the track.

3.  **Ambiguity Handling:** If the evidence is plausible but not conclusive (e.g., multiple agents could have made the journey), the `IdentityResolver` does not force a decision. Instead, it marks the track with a set of `identity_candidates`.

4.  **New Agent Creation:** If a track cannot be plausibly linked to any existing agent, a new `GlobalAgent` is created.

## Output

The `IdentityResolver` outputs `LocalTrack`s that now have either:
- A confirmed `global_agent_id` (if merged).
- A set of `identity_candidates` (if ambiguous).
- A new `global_agent_id` (if no match was found).

This data is then ready for final resolution by the `EvidenceProcessor` or for storage in the database.

---

## Advanced Logic: Portals and Uncertainty

The system models the world as a set of nested containers (Portals) to handle concealment.

### 1. Handling Indefinite Concealment (Structure Portals)
*   **Concept:** A "Structure Portal" (e.g., a house) allows an agent to disappear for any duration.
*   **Logic:** When linking across a portal, the Grid's *maximum* time limit is ignored, but the *minimum* travel time remains a hard constraint.
*   **Group Association:** If a group enters a portal and a same-sized group exits, and individual identities are ambiguous, the system assigns the *entire set of candidates* to all exiting tracks. It does not guess who is who.

### 2. Handling Mobile Containers (Vehicle Portals)
*   **Concept:** A vehicle is a mobile portal. It has a state defined by its `PotentialOccupants`.
*   **The "Trojan Horse" (Uncertain Arrival):** If a vehicle arrives with unknown occupants, its state is flagged as `UNCERTAIN`. This tells the system that the list of observed occupants is incomplete.
*   **Possibility Set Inheritance:** When an agent enters a portal (Garage) that contains a vehicle (Car), the agent is added to the vehicle's `PotentialOccupants` set.
*   **Process of Elimination:** When the system confirms an agent is *outside* a vehicle (e.g., via a face match on a pedestrian track), it removes that agent from the vehicle's `PotentialOccupants` set. This subtractive logic is the primary way ambiguity is resolved.

---

## Post-Processing and Maintenance

### Agent Merging (Resolving Split-Brain)

Over the long term, the system may mistakenly create two separate `GlobalAgent` IDs for the same individual (a "split-brain" scenario). A mechanism is required to correct this. The system shall provide an administrative function, `MergeAgents(primary_agent_id, secondary_agent_id)`, which performs the following actions:
1.  Reassigns all `LocalTrack`s from the `secondary_agent_id` to the `primary_agent_id`.
2.  Copies any relevant metadata (like face crops) from the secondary to the primary agent.
3.  Archives the `secondary_agent_id` to prevent its future use, while maintaining a record of the merge for historical integrity.

This is a permanent data correction operation, not a state change, and is essential for maintaining a clean identity database.

### Group Association (Handling Ambiguous Exits)

When a group of agents enters a portal and a same-sized group exits, it can be impossible to resolve individual identities if clear visual evidence is unavailable. In this case, the `IdentityResolver` must not guess.

Instead, it will apply a **Group Association**. It will assign the *same set of candidate IDs* to all the exiting tracks. For example, if P1 and P2 enter a portal, and two unidentifiable tracks (`Track_A`, `Track_B`) exit, their state will be:
-   `Track_A.identity_candidates = {P1, P2}`
-   `Track_B.identity_candidates = {P1, P2}`

This correctly captures the ambiguity of the situation and queues both tracks for review, preventing a potential identity swap.
