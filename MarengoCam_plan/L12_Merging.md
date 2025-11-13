# Level 12 - Merging Technical Specs

## Purpose
Technical implementation of cross-camera merge logic.

## Merge Algorithm

### Candidate Finding
**Runs periodically (every 30s):**
```python
def find_merge_candidates():
    recent_tracks = get_tracks(last_60_seconds)
    for track1 in recent_tracks:
        for track2 in recent_tracks:
            if time_space_plausible(track1, track2):
                if portal_connected(track1.camera, track2.camera):
                    yield (track1, track2)
```

### Time/Space Gating
**Filters impossible transitions:**
- Temporal: `track2.start_time - track1.end_time < max_window`
- Spatial: distance / time_delta within walking speed (~1 m/s)
- Portal check: track endpoints near portal locations

### Evidence Scoring
**Count evidence, threshold:**
```python
strong_evidence = 0
weak_evidence = 0

if face_match(track1, track2) > 0.75:
    strong_evidence += 1
if only_candidate(track2):
    strong_evidence += 1
if portal_crossing(track1, track2):
    strong_evidence += 1
if spatial_proximity(track1, track2):
    weak_evidence += 1

if strong_evidence >= 2:
    return AUTO_MERGE
elif strong_evidence >= 1 and weak_evidence >= 1:
    return REVIEW_LLM
else:
    return REVIEW_HUMAN
```

## Face Recognition

### CodeProject.AI Integration
**Endpoint:**
```
POST http://{codeproject_server}/v1/vision/face/recognize
```

**Request:**
```json
{
  "image": "base64_encoded_jpeg",
  "min_confidence": 0.60
}
```

**Response:**
```json
{
  "predictions": [
    {
      "userid": "person_a",
      "confidence": 0.85
    }
  ]
}
```

### Face Library Management
- Register face: POST to `/v1/vision/face/register`
- Delete face: DELETE `/v1/vision/face/{userid}`
- List faces: GET `/v1/vision/face/list`

## LLM Adjudication

### Gemini Flash 2.0
**Endpoint:** Google AI Studio API

**Prompt template:**
```
Are these the same person/vehicle?

Image Set A:
{panel_a_images}

Image Set B:
{panel_b_images}

Context:
- Camera A → Camera B
- Time gap: {time_delta} seconds
- Portal: {portal_name}

Respond in JSON:
{
  "answer": "YES|NO|CANNOT",
  "confidence": 0.0-1.0,
  "reasons": "brief explanation"
}
```

### Panel Assembly
**2×6 layout (768×768 pixels):**
- Top row: Track A best frames (K=3)
- Bottom row: Track B best frames (K=3)
- Quality scoring: blur, brightness, face-visible

## Multi-Assignment

### Data Structure
**Track can link to multiple agents:**
```python
class Track:
    agent_assignments: List[AgentAssignment]

class AgentAssignment:
    agent_id: int
    confidence: float  # 0.0-1.0
    evidence: List[str]  # ["face_match", "portal_crossing"]
    validated: bool
```

### Uncertainty Persistence
- Assignments persist until validated (human/LLM)
- Timeline queries return all possibilities
- UI shows uncertainty ("possibly Agent A or B")

## Related Documents
- **Source:** AGENT_TRACKING_AND_MERGE.md (deprecated)
- **L3_Merging:** Non-technical approach
- **L4_Complexity:** Where this gets hard
