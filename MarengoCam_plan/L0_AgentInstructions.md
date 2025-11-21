# Level 0 - Operating System

## Purpose
This document provides instructions for AI agents working with the MarengoCam planning documentation. It defines the structure, conventions, and workflows for maintaining project coherence.

## Documentation Hierarchy

### Level Structure

**Strategic Levels (L0-L6):**
- **Level 0**: This document - meta-instructions for agents (evolves continuously)
- **Level 1**: Mission and philosophy (core problem, values, goals)
- **Level 2**: Architecture and strategy (components, data structures, what and why)
- **Level 3**: Tactics (component behavior, how and when)
- **Level 4**: Concepts (detailed specs for complex ideas referenced by L3)
- **Level 5**: Bootstrap strategy (concurrent learning workflow)
- **Level 6**: Deployment plan (phased rollout, gates, risks)

**Technical Specification Levels (L{N}T):**
- `L{N}T` files contain technical specifications (e.g., algorithms, data structures, APIs) for the concepts introduced in the corresponding `L{N}` document.
- **L1T**: Terminology and nomenclature reference (the project "dictionary").
- **L2T**: Technical architecture specifications.
- **L3T**: Implementation specifications for L3 tactical components.
- **L4T**: Implementation specifications for L4 concepts.

**Pattern:** `L{N}` (Strategic/Tactical) → `L{N}T` (Technical Specification)

**Code:** Lives outside this framework (separate repo/directory)

### Naming Conventions
- Core documents: `L{N}_{Topic}.md` or `L{N}T_{Topic}.md`
  - Example: `L1_Mission.md`, `L2T_Architecture.md`, `L3_EvidenceProcessing.md`
- Discourse documents: `L{N}D_{Topic}.md` or `L{N}TD_{Topic}.md`
  - Example: `L1D_Mission.md`, `L3TD_Database.md`
- Avoid redundant words in filenames (e.g., use `L3_Tracking.md` not `L3_Tactics_Tracking.md`)

### Discourse Files (D-files)
- Paired with each level file
- Contain brief rationale for decisions and tradeoffs
- Keep entries concise - record of discourse, not exhaustive analysis
- Track key decision points and alternatives considered

## Standard Operating Procedure

To ensure clarity and prevent context drift, you will adhere to the following collaborative cycle for any major objective.

### Task Management with TODO Lists

For any multi-step objective, we will liberally use a visible TODO list to track progress.

## Current Phase: Refactoring to Component Specs (GeminiProposal2)

**Objective:** Transition from the fragmented `Lx`/`LxT` structure to a "Buildable" Component Specification structure (`00` through `05`).

**Philosophy:**
- **Consolidation:** Merge "Human" (Why) and "Technical" (How) into single Component Specs.
- **Precision:** Move from "we need a table" to `CREATE TABLE`. Move from "we need a class" to `class Interface:`.
- **Completeness:** Ensure every component has a defined Input, Process, Output, and Configuration.

**Roadmap:**

1.  **00_Architecture.md (The Map)**
    - Define the high-level Hub-and-Spoke graph.
    - Define the "Event Bus" vs "Direct Call" boundaries.
    - *New:* Define the Deployment View (Docker containers).

2.  **01_DataModel.md (The Vocabulary)**
    - **Schema:** Full SQL DDL for `GridLink`, `Evidence`, `LocalTrack`, `CameraPairSummary`.
    - **Types:** Pydantic/JSON schemas for inter-service messages.
    - **Storage:** Define exactly *where* each piece lives (Postgres vs Redis vs In-Memory).

3.  **02_Component_IdentityResolver.md (The Brain)**
    - **Algorithm:** Exact steps for `resolve(track)`.
    - **Logic:** The "Triangle Inequality" rejection logic (from Stress Tests).
    - **Logic:** The "Bloom Filter" fast-fail logic.

4.  **03_Component_ChunkProcessor.md (The Worker)**
    - **State Machine:** Lifecycle of a `Chunk` (Recording -> Processing -> Evidence -> Archiving).
    - **Interface:** Inputs (BlueIris, MQTT) and Outputs (Evidence DB).

5.  **04_Component_SystemHealth.md (The Immune System)**
    - **Survival Mode:** Concrete triggers (e.g., "Queue > 100 items", "Lag > 5s").
    - **Actions:** Specific load-shedding steps (e.g., "Drop B-frames", "Disable Face Rec").
    - **Metrics:** List of specific metrics to export.

6.  **05_Configuration.md (The Controls)**
    - **Schema:** Full `config.yaml` structure.
    - **Secrets:** Handling of passwords/keys.

### Step 1: Define the Objective

Before beginning a new phase of work, you must explicitly define the high-level goal with the user.
- Propose a clear, single-sentence objective (e.g., "The current objective is to align all L3 documents with the L2 strategy.").
- Await user approval or amendment before proceeding.

### Step 2: Execute in a Strict Loop

For every file modification or significant action, you must follow the **"Read -> Recommend -> Approve -> Act"** cycle.
1.  **Read:** Gather all necessary context and information.
2.  **Recommend:** Present a concise analysis and a clear recommendation. **You must not take any action at this stage.**
3.  **Approve:** Await explicit user approval (e.g., "yes," "agreed," "do it").
4.  **Act:** Only after approval, execute the specific action.

### Step 2a: Consensus Protocol for Design Decisions

When the user provides a design insight or specification idea, you must **ask clarifying questions first** before implementing any significant work (>50 lines of code/documentation).

**Required workflow:**
1. **User provides insight** (e.g., "the ID has to be something like 'whoever is in track xyz' so there's a trail")
2. **You translate to specific structure** - "So the track_id should be `<camera>-<object_type>-<grid_cell>-<time_range>`, and we store a full IdentityDecision history with each elimination step?"
3. **You ask clarifying questions** - "Is that the structure you're envisioning? Anything else? Should we also track X?"
4. **User confirms or redirects** - "Yes, but also..." OR "No, I meant..."
5. **Only then proceed** - Once you have explicit confirmation, you can write the specification/code

**Why:** Prevents writing 400 lines assuming you understand the intent, when you may have misunderstood a critical aspect. The approval is visible in the conversation itself, not buried in tool status.

**When to apply:** Any time you're about to create or modify files that implement a user insight, regardless of size. This includes:
- New files (any size)
- Edits that add new fields, data structures, or concepts
- Changes to existing specifications that extend or reinterpret them
- Integration work that connects multiple concepts together

Do NOT apply this for:
- Reading/researching/exploring (information gathering only)
- Asking clarifying questions (always fine)
- Reverting/undoing changes
- Fixing obvious errors or typos
- Reformatting/refactoring without changing meaning

**Key principle:** If your change could represent a design decision—even small ones—ask first. Big ideas come with few words, and small edits can encode significant assumptions.

### Step 3: Checkpoint and Commit

Upon completion of the defined objective, you must formally checkpoint the work.
- Propose a `git commit` with a descriptive message summarizing the accomplishment.
- This concludes the cycle and provides a clean state before defining the next objective.

## File Organization Rules

### When to Split Files
Split a level into multiple files when:
- Topic becomes too large to review effectively
- Component has distinct scope (e.g., Ingestor, EventInference, DataStorage)
- Naming pattern: `L{N}_{Component}.md` + `L{N}D_{Component}.md`

### Directory Structure
- **Root**: All active Level files and D-files (L0-L6, L{N}T, L{N}D)
- **deprecated/**: Old documentation being replaced
- **decisions/**: Architecture Decision Records (ADRs) - specific technical choices
  - Example: YOLO11_MIGRATION_ANALYSIS.md, PREPOPULATION_STRATEGY.md
  - Not part of hierarchy, but preserved for context

## Workflow for Agents

### 1. Review Process
- Start with L1 (Mission)
- Push changes through all levels
- Identify conflicts and inconsistencies
- Iterate until coherent

### 2. Editing Protocol
- Discourse with user before major changes
- Maintain consistency across levels
- Update corresponding D-files with decision rationale
- Ensure technical specifications (L{N}T files) align with their corresponding L{N} files.

### 3. Consistency Requirements
- Each level must be internally consistent
- Lower levels must support higher levels
- Technical specs must implement tactical plans
- Avoid contradictions between parallel documents at same level

## Agent Behavior Guidelines
- Be fast and insightful
- Collaborative, not stubborn
- Defer to user judgment on disagreements
- Help refine and execute, don't override
- Surface conflicts and inconsistencies proactively

## Writing Style: Clarity and Brevity

**Core principle:** Extra content sends agents down rabbitholes for no reason.

**Do:**
- Use observable, testable success criteria
- Write "what must work" not "what we might do"
- Sequence matters, timelines don't (Phase 0 → Phase 1, not "Week 1-2")
- Keep files focused and minimal

**Don't:**
- Make up timelines ("Week 1-2", "Months 3-4")
- Enumerate vague tools ("configuration viewer", "statistics dashboard")
- Create metrics tables with fake percentages
- Add "Related Documents" sections with broken references

**Examples:**

**Good:**
```
## Phase 1: Single-Camera Tracking
**Goal:** LocalTracks work on one camera
**Success Gate:** Track person across multiple 12s chunks
```

**Bad:**
```
## Phase 1: Single-Camera Tracking (Weeks 3-4)
**Goal:** Prove intra-camera tracking before cross-camera merging
**Tasks:** YOLO11 tracking integration, camera state machine, track quality filtering, agent creation
**Tools:** Single-camera debug viewer, state transition logger
**Success:** Track person for 60+ seconds, state transitions work
```

**Why bad:**
- Timeline is voodoo ("Weeks 3-4")
- Task list is obvious from goal
- Tool list doesn't add value
- Success criteria could be sharper

## Scope Guidelines by Level

### Level 1 (Mission)
**What belongs:** Core problem statement, values/philosophy, high-level goals
**What doesn't:** Specific tools, methods, technical details, timelines, phasing

### Level 2 (Strategy)
**What belongs:** Non-technical approach (track merging, evidence-based decisions)
**What doesn't:** Specific APIs, thresholds, code-level details

### Level 3 (Tactics)
**What belongs:** How components work together, when things happen
**What doesn't:** Code implementation, database schema

### Level 4 (Concepts)
**What belongs:** Detailed specs for single concepts (ReviewPanel, GridLearning, AlibiCheck)
**What doesn't:** Implementation code, cross-cutting concerns

### Level 6 (Deployment)
**What belongs:** Sequence, gates, risks. Observable success criteria per phase.
**What doesn't:** Timelines (voodoo), tool lists (obvious), metrics tables (fake data)

### L{N}T Files (Technical Specifications)
**What belongs:** APIs, data structures, algorithms, schemas, and other technical specifications that provide concrete implementation details for the concepts described in the corresponding `L{N}` files.
**What doesn't:** Philosophy or strategy (that belongs in L1-L3).

## Version Control
- **Level 0 evolves continuously** - rewrite as framework clarifies
- Refinements to structure should be discussed and approved
- This document reflects current understanding of hierarchy
