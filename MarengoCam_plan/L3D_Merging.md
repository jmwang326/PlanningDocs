# Level 3 Discourse - Merging - Policy vs. Mechanism

## Decision
The system will separate fact-gathering from decision-making, following a "Policy vs. Mechanism" pattern.
- **Evidence Engine (Mechanism):** Gathers facts and creates an `EvidenceReport`.
- **Track Merging Engine (Policy):** A pure rules engine that receives the report and makes a decision.

## Rationale
This separation improves clarity, testability, and extensibility. The decision logic (`Track Merging Engine`) remains clean and readable, while the data I/O complexity is isolated within the `Evidence Engine`. Adding new evidence types only requires updating the "Mechanism" and adding a simple rule to the "Policy".