# Level 3 Discourse - Inference Manager - Human as a Provider

## Decision
The Human-in-the-Loop (HITL) review process will be treated as a specialized "provider" endpoint within the `Inference Manager`, not as a separate architectural component.

## Rationale
This simplifies the architecture. The `Inference Manager` can route "adjudication" tasks to either an LLM or a human reviewer using the same unified interface and priority system. The "Human Provider" client will simply add the task to the database review queue, which the GUI reads from. This avoids creating a separate, complex workflow for human reviews.