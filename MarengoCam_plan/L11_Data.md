# Level 11 - Data Architecture

## Purpose
Technical specifications for data structures, storage, and schemas.

## Database Schema

### Core Tables
- **agents:** Persistent entities (people, vehicles)
- **tracks:** Observation segments per camera
- **detections:** Frame-level bounding boxes
- **merges:** Track assignments to agents
- **validations:** Human/LLM merge decisions

### Metadata Storage
**SQLite database** (single-user, embedded)
- Metadata persists indefinitely
- Frame files pruned after retention period
- Learned data (face library, portal timing) preserved

## Frame Storage

### Directory Structure
```
storage/
  frames/
    {camera_id}/
      {date}/
        {timestamp}.jpg
  metadata/
    marengo.db
  state/
    face_library/
    portal_stats/
```

### Manifest Format
**JSON per event:**
- Frame list with timestamps
- Track metadata
- Agent assignments
- Reconstruction parameters

## Learned Data

### Face Library
**CodeProject.AI format:**
- Face embeddings (512-dim)
- Linked to agent identities
- Quality scores per embedding

### Portal Timing Statistics
**Per portal:**
- Typical_time (mean, median, P50)
- Max_time (P90 or user-configured)
- Observation count
- Last updated timestamp

## Related Documents
- **Source:** DATA_AND_STORAGE_ARCHITECTURE.md (deprecated)
- **L2 (Strategy):** Why this data model
- **L3 (Tactics):** How data flows through system
