# Level 4 - Timeline Viewer

## Purpose
Specification for the timeline viewer application. This is the primary interface for browsing, filtering, and replaying the reconstructed timeline of an agent's activity across all cameras.

**Referenced from:** [L3_Gui.md](L3_Gui.md)

**Implemented in:** L14_TimelineUI.py

---

## UI Layout

```
┌───────────────────────────────────────────────────────────────────────────┐
│ MarengoCam Timeline Viewer                                                │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│ ┌──────────────┬────────────────────────────────────────────────────────┐ │
│ │ AGENTS (12)  │ FILTERS: [Date Range] [Camera ▼] [Confidence ▼] [Search] │ │
│ │              ├────────────────────────────────────────────────────────┤ │
│ │ Agent A      │ Timeline for: Agent A (14 events)                      │ │
│ │ Agent B      │                                                        │ │
│ │ Agent C (✓)  │ ┌────────────────────────────────────────────────────┐ │ │
│ │ Agent D      │ │ ┌────────────────────────────────────────────────┐ │ │ │
│ │ ...          │ │ │                VIDEO PLAYER                  │ │ │ │
│ │              │ │ │                                                │ │ │ │
│ │              │ │ │       (Click an event to play video)           │ │ │ │
│ │              │ │ │                                                │ │ │ │
│ │              │ │ └────────────────────────────────────────────────┘ │ │ │
│ │              │ │                                                    │ │ │
│ │              │ │ 10:35:12 - FrontDoor (6.2s) [Confidence: High]     │ │ │
│ │              │ │    - Merged from Driveway (4.1s travel time)       │ │ │
│ │              │ │                                                    │ │ │
│ │              │ │ 10:34:48 - Driveway (8.1s) [Confidence: High]      │ │ │
│ │              │ │    - Face match: 0.82                              │ │ │
│ │              │ │                                                    │ │ │
│ │              │ │ 10:33:15 - Backyard (12.0s) [Confidence: Uncertain]│ │ │
│ │              │ │    - Candidates: Agent C, Agent F                  │ │ │
│ │              │ │    - [Review this merge]                           │ │ │
│ │              │ └────────────────────────────────────────────────────┘ │ │
│ └──────────────┴────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### 1. Agent List Panel
- **Purpose:** To select which agent's timeline to view.
- **Content:** A searchable, scrollable list of all known agents (both named and unnamed, e.g., "Agent 123").
- **Functionality:**
    - Displays agent name/ID and a representative thumbnail.
    - Search box to filter agents by name/ID.
    - Indicates the currently selected agent.

### 2. Filter Bar
- **Purpose:** To refine the events shown in the timeline view.
- **Controls:**
    - **Date Range Picker:** Select start and end dates/times.
    - **Camera Dropdown:** Filter events by one or more cameras.
    - **Confidence Filter:** Show all events, or only `High` or `Uncertain` confidence events.
    - **Search Box:** Free-text search on event details (e.g., "portal", "vehicle").

### 3. Video Player
- **Purpose:** To play the video clip associated with a selected timeline event.
- **Functionality:**
    - Displays the 12-second video chunk for the selected `LocalTrack`.
    - Highlights the bounding box of the agent within the video.
    - Standard video controls: play, pause, scrub.
    - Displays a "Click an event to play video" message by default.

### 4. Timeline Event View
- **Purpose:** To display the chronological list of events (`LocalTracks`) for the selected agent.
- **Content:** A scrollable list of event cards, ordered from most recent to oldest.
- **Each Event Card Contains:**
    - **Primary Info:** Start time, Camera ID, and duration of the track.
    - **Confidence Level:** `High` (auto-merged with strong evidence) or `Uncertain` (queued for review or has unresolved candidates).
    - **Evidence Snippet:** The key piece of evidence used for the merge (e.g., "Face match: 0.82", "Grid overlap", "Human validated").
    - **Context:** For uncertain events, lists the other `identity_candidates`.
    - **Action Link:** A `[Review this merge]` link for uncertain events, which navigates to the Review Panel for that track.

---

## User Workflows

### Viewing a Timeline
1. User selects an agent from the **Agent List**.
2. The **Timeline Event View** populates with all events for that agent.
3. The user scrolls through the chronological history.

### Playing a Video Clip
1. User clicks on an event card in the **Timeline Event View**.
2. The corresponding video loads and plays in the **Video Player**.
3. The selected event card is highlighted.

### Filtering the Timeline
1. User selects a date range or a camera from the **Filter Bar**.
2. The **Timeline Event View** updates in real-time to show only the matching events.

### Investigating an Uncertain Merge
1. User filters the timeline to show only `Uncertain` events.
2. User clicks the `[Review this merge]` link on an event card.
3. The application navigates to the **Review Panel** UI for that specific track, allowing the user to resolve the uncertainty.

---

## API Dependencies

This UI will be powered by the following API endpoints defined in `L3_Gui.md`:

- `GET /api/agents`: To populate the **Agent List Panel**.
- `GET /api/agents/{agent_id}/timeline`: To fetch the events for the **Timeline Event View**. This endpoint should support the filtering parameters.
- `GET /api/tracks/{track_id}/video`: To stream the video for the **Video Player**.
