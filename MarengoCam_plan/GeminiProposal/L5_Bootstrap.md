# L5: Bootstrap Strategy

## Component: `Bootstrap`

## Purpose

The Bootstrap component defines the strategy for rapidly advancing the system's intelligence from a "cold start" to a highly autonomous state. The core philosophy is to deploy a functional, baseline system immediately and then use dedicated tools to build its knowledge in parallel with live operation.

This approach avoids a lengthy, offline training period. Instead, the system learns from real-world data, curated by a human-in-the-loop, from day one.

## High-Level Concepts

### Concurrent Learning and Operation

The system does not need to be "smart" before it is useful. The initial deployment (Phase 1) focuses on robustly tracking objects and queueing all uncertain decisions for human review. In parallel, a human operator uses specialized GUI tools to perform two critical functions:

1.  **Build the Face Library:** By identifying a small number of key individuals, the operator provides the system with its most powerful tool for automatically merging identities.
2.  **Curate Training Data:** By validating system decisions, the operator simultaneously generates high-quality, labeled data for future Re-ID model fine-tuning.

### The Learning Curve: An Exponential Handover

The bootstrap process is designed as an exponential handover from human to machine.

-   **Initially (Days 1-14):** The human's role is intensive. They review most of the system's decisions, effectively teaching it "who's who" and what correct associations look like.
-   **As the system learns (Days 15-30+):** The system's auto-merge capabilities (driven by face and grid learning) increase dramatically. The human's workload shifts from constant review to only handling rare, complex scenarios that the machine cannot confidently resolve.

The goal is to transition from a manually-assisted system to a primarily autonomous one within approximately 30 days.

### Transitioning to Autonomy

The system's operational mode is explicitly configured to reflect its maturity.

1.  **`PROD_LEARNING` Stage:** A conservative initial state where the system prioritizes clean data collection and relies heavily on human validation.
2.  **`PROD_AUTO` Stage:** The mature, autonomous state where the system operates with high confidence, leveraging its learned models to minimize human intervention.

The transition between these stages is triggered by meeting specific, measurable metrics that prove the system's readiness.

## Required Tooling

To facilitate this human-in-the-loop learning process, a suite of dedicated tools is required. These tools allow the operator to efficiently review decisions, manage the face library, and monitor the system's learning progress.
