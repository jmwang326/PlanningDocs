# Level 0 - Operating System

## Purpose
This document provides instructions for AI agents working with the MarengoCam planning documentation. It defines the structure, conventions, and workflows for maintaining project coherence.

## Documentation Hierarchy

### Level Structure

**Strategic Levels (L1-L9):**
- **Level 0**: This document - meta-instructions for agents (evolves continuously)
- **Level 1**: Mission and philosophy (core problem, values, goals)
- **Level 2**: Strategy (what and why, non-technical approach)
- **Level 3**: Tactics (how and when, non-technical, component-specific)
- **Level 4**: Complexity and risks (what's hard, tuning-dependent, failure modes)
- **Level 5**: TBD (open for future strategic topics)
- **Level 6**: Deployment plan (phased rollout, timelines, success criteria)

**Reference Levels (L10):**
- **Level 10**: Nomenclature (terminology reference, the "dictionary")

**Technical Levels (L11-L19):**
- **Level 11**: Technical specs corresponding to L1 topics
- **Level 12**: Technical specs corresponding to L2 topics
- **Level 13**: Technical specs corresponding to L3 topics
- **Level 14**: Technical specs corresponding to L4 topics
- **Level 15**: Technical specs corresponding to L5 topics
- **Level 16**: Technical specs corresponding to L6 topics

**Pattern:** L{N} strategic → L{N+10} technical

### Naming Conventions
- Core documents: `L{N}_{Topic}.md`
  - Example: `L1_Mission.md`, `L2_Strategy.md`, `L3_Tracking.md`
- Discourse documents: `L{N}D_{Topic}.md`
  - Example: `L1D_Mission.md`, `L3D_Tracking.md`
- Avoid redundant words in filenames (e.g., use `L3_Tracking.md` not `L3_Tactics_Tracking.md`)
- Level 10 is special: `L10_Nomenclature.md` (standalone terminology reference)

### Discourse Files (D-files)
- Paired with each level file
- Contain brief rationale for decisions and tradeoffs
- Keep entries concise - record of discourse, not exhaustive analysis
- Track key decision points and alternatives considered

## Standard Operating Procedure

To ensure clarity and prevent context drift, you will adhere to the following three-step collaborative cycle for any major objective.

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
- **Root**: All active Level files and D-files (L0-L13, L{N}D)
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
- Ensure technical levels (11-13) align with tactical levels (2-3)

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

### Level 6 (Deployment)
**What belongs:** Phased rollout, timelines, success criteria per phase
**What doesn't:** Implementation tasks (that's project management)

### Level 11-13 (Technical)
**What belongs:** APIs, data structures, algorithms, schemas, code specs
**What doesn't:** Philosophy or strategy (that's L1-L3)

## Version Control
- **Level 0 evolves continuously** - rewrite as framework clarifies
- Refinements to structure should be discussed and approved
- This document reflects current understanding of hierarchy
