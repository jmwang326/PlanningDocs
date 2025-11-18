# L0: Meta-Documentation

This document outlines the philosophy and structure of the MarengoCam planning documents. The primary goal is to create a dual-audience documentation system that is both human-readable and optimized as an implementable blueprint for intelligent agents.

## Core Principles

1.  **Audience-Specific Documents:** A clear distinction is maintained between documents intended for human understanding and those formatted for agent implementation.
2.  **Concise Logic:** Human-facing documents should explain logic and behavior clearly and concisely, avoiding implementation-specific jargon.
3.  **Unambiguous Specification:** Agent-facing documents must provide precise, unambiguous, and machine-readable details required for coding.
4.  **Single Source of Logic:** The human-facing `L` file is the single source of truth for a component's intended behavior. The corresponding `LT` file is a direct translation of that logic and must not introduce new concepts.

## Document Structure

The documentation is organized into two main types of files, distinguished by their audience and purpose:

### `L{level}_{topic}.md` (Human-Readable Logic)

-   **Audience:** **Humans** (Developers, Architects, Stakeholders).
-   **Purpose:** To serve as the single source of truth for the *logic* and *intended behavior* of a system component.
-   **Content:** Describes the system's logic in clear, narrative prose. The primary goal is to be understood by human readers, ensuring they are not overwhelmed with technical implementation details. All core logic resides here.

### `L{level}T_{topic}.md` (Technical Specification)

-   **Audience:** **AI Coding Agents** and developers during active implementation.
-   **Purpose:** To translate the logic from the corresponding `L` file into a precise, implementable technical specification.
-   **Content:** This file does not introduce new logic. It provides the technical "how" by detailing function contracts, data structures, and other specifications derived directly from the `L` file's "what."
-   **Dependency Manifest:** To be fully machine-actionable, each `LT` file **must** begin with a "Dependency Manifest" that explicitly lists all other documents and configuration sections it depends on. This creates a self-contained context for an implementation task.
