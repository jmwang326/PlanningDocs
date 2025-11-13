# MarengoCam Planning Documentation

## Quick Start

**New to this project?** Read in this order:
1. [Level0.md](Level0.md) - Framework and navigation guide
2. [L1_Mission.md](L1_Mission.md) - Core problem and philosophy
3. [L2_Strategy.md](L2_Strategy.md) - High-level approach
4. [L6_Deployment.md](L6_Deployment.md) - Phased rollout plan

## Documentation Hierarchy

### Core Planning
- **[Level0.md](Level0.md)** - Meta-instructions for agents (evolves continuously)
- **[L1_Mission.md](L1_Mission.md)** - Mission and philosophy
- **[L2_Strategy.md](L2_Strategy.md)** - Strategy (what and why)
- **[L3_Acquisition.md](L3_Acquisition.md)** - Acquisition tactics
- **[L3_Detection.md](L3_Detection.md)** - Detection and tracking tactics
- **[L3_Merging.md](L3_Merging.md)** - Cross-camera merging tactics
- **[L4_Complexity.md](L4_Complexity.md)** - Complexity and risks
- **[L6_Deployment.md](L6_Deployment.md)** - Deployment plan

### Reference
- **[L10_Nomenclature.md](L10_Nomenclature.md)** - Terminology dictionary

### Technical Specifications
- **[L11_Data.md](L11_Data.md)** - Data architecture
- **[L11_Acquisition.md](L11_Acquisition.md)** - Acquisition technical specs
- **[L12_Merging.md](L12_Merging.md)** - Merging technical specs

### Discourse Files (D-files)
- **[L1D_Mission.md](L1D_Mission.md)** - Mission decisions and tradeoffs
- **[L2D_Strategy.md](L2D_Strategy.md)** - Strategy decisions and tradeoffs

## Directory Structure

```
MarengoCam_plan/
├── Level0.md                    # Framework OS
├── L1_Mission.md                # Mission
├── L2_Strategy.md               # Strategy
├── L3_*.md                      # Tactics (by component)
├── L4_Complexity.md             # Complexity/risks
├── L6_Deployment.md             # Deployment plan
├── L10_Nomenclature.md          # Terminology
├── L11_*.md                     # Technical specs
├── L12_*.md                     # Technical specs
├── L*D_*.md                     # Discourse files
├── deprecated/                  # Old documentation
│   ├── BIGPICTURE_V2.md
│   ├── AGENT_TRACKING_AND_MERGE.md
│   └── ...
└── decisions/                   # Architecture Decision Records
    └── (ADRs TBD)
```

## Framework Philosophy

This documentation follows a **hierarchical framework**:
- **L0:** Operating system (meta-instructions)
- **L1:** Mission (problem, values, goals)
- **L2:** Strategy (what and why, non-technical)
- **L3:** Tactics (how and when, non-technical)
- **L4:** Complexity (what's hard)
- **L6:** Deployment (phased rollout)
- **L10:** Nomenclature (terminology)
- **L11-13:** Technical specifications

**D-files** track discourse (brief rationale for decisions).

See [Level0.md](Level0.md) for detailed framework rules.

## Status

**Current Phase:** Framework reorganization in progress

**Completed:**
- L0, L1, L2 (Mission and Strategy defined)
- L3 (Tactics sketched)
- L4 (Complexity identified)
- L6 (Deployment plan outlined)
- L10 (Nomenclature migrated)
- L11-12 (Technical specs started)

**Next Steps:**
- Review L1-L6 for consistency
- Expand L11-13 technical specifications
- Iterate based on conflicts

## Old Documentation

**Deprecated files** moved to `deprecated/` directory:
- BIGPICTURE_V2.md
- AGENT_TRACKING_AND_MERGE.md
- MASTER_ROLLOUT.md
- And others...

These are preserved for reference but **superseded by new hierarchy**.
