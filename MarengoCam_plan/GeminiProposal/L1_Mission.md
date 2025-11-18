# L1: Mission

## Core Objective

To solve the multi-camera assignment problem by transforming disconnected observations from multiple cameras into a unified, coherent timeline for each tracked agent.

## Problem Statement

- **Input:** Disconnected `LocalTrack` objects from individual cameras.
- **Core Question:** Do `LocalTrack_A` from `Camera_1` and `LocalTrack_B` from `Camera_2` represent the same `GlobalAgent`?
- **Output:** A unified timeline of an agent's activity across the property (e.g., "Agent 123 entered via Driveway, crossed to FrontDoor, and exited through Gate").

## System Mandate

This is a **forensic reconstruction system**, not a real-time alerting system.

- **Primary Use Case:** Answer the question, "Is there an unknown person on the property, and what was their path?"
- **In Scope:**
    - Detecting and merging tracks to build a `GlobalAgent` timeline.
    - Identifying known agents via face recognition.
    - Handling uncertainty through evidence-based resolution.
- **Out of Scope:**
    - Real-time alarm generation.
    - Behavioral prediction.

## Success Criteria

1.  **Timeline Accuracy:** The reconstructed timeline must accurately reflect the ground truth of an agent's movement.
2.  **Identity Accuracy:** Known agents are correctly identified.
3.  **Low Error Rate:** False merge rate must be below 2%.
4.  **Automation:** Achieve >80% auto-merge rate after the initial bootstrapping/learning phase.
