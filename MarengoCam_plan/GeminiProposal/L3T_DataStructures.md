# L3T: Core Data Structures

This document provides the Python `dataclass` definitions for the primary data structures used throughout the MarengoCam system. These definitions serve as the in-memory representation of the data stored in the database.

---

## `LocalTrack`

This is the fundamental data structure of the entire system. It represents a single, continuous observation of an agent on one camera within one chunk.

```python
from dataclasses import dataclass, field
from typing import Optional, List, Tuple, Set

@dataclass
class LocalTrack:
    """A continuous observation of one agent on one camera in one chunk."""

    # Core Identity & Database PK
    id: Optional[int] = None
    camera_id: str = ""
    chunk_id: int = 0
    local_id: int = 0 # The tracker's ID within the chunk

    # Resolved Identity
    global_agent_id: Optional[int] = None # NULL if uncertain

    # Timing
    start_time: float = 0.0
    end_time: float = 0.0

    # Spatial Information (6x6 Grid)
    entry_cell: Optional[Tuple[int, int]] = None
    exit_cell: Optional[Tuple[int, int]] = None

    # Relationships
    previous_track_id: Optional[int] = None
    next_track_id: Optional[int] = None

    # Uncertainty
    identity_candidates: Set[int] = field(default_factory=set)

    # Contextual Information
    exit_reason: Optional[str] = None # "left_frame", "chunk_boundary", "portal_crossed"
    at_portal: Optional[str] = None   # ID of the portal, if applicable

    # Raw Data (from JSONB)
    detections: List['Detection'] = field(default_factory=list)
```

---

## `Detection`

A single bounding box detection from a single frame. A `LocalTrack` is composed of a list of `Detection`s.

```python
@dataclass
class Detection:
    """A single bounding box from a single frame of video."""
    frame_time: float
    bbox_x: int
    bbox_y: int
    bbox_w: int
    bbox_h: int
    confidence: float
    class_name: str
```

---

## `Evidence`

A structure for passing new information into the `EvidenceProcessor` to resolve uncertainty.

```python
@dataclass
class Evidence:
    """A package of new information for resolving track uncertainty."""
    track_id: int
    source: str  # "face_recognition", "human_review", "llm"

    # The decision based on the evidence
    decision: str  # "merge" or "reject"
    agent_id: int  # The agent_id to merge with or reject

    # Optional metadata
    confidence: Optional[float] = None
    reasoning: Optional[str] = None
```
