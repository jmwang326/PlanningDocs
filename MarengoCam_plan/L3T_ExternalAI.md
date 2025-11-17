# Level 13 - External AI API Contracts

## Purpose
API contracts for external AI services (CodeProject.AI, LLM). Request/response formats, error handling, retry logic.

---

## CodeProject.AI - Face Recognition

**Service:** Face recognition with trainable library

**Base URL:** Configured in `marengo.yaml` (e.g., `http://localhost:32168`)

---

### Register Face

**Endpoint:** `POST /v1/vision/face/register`

**Request:**
```json
{
  "image": "<base64_encoded_jpeg>",
  "userid": "agent_123"
}
```

**Response (Success):**
```json
{
  "success": true,
  "message": "face registered"
}
```

**Response (Error):**
```json
{
  "success": false,
  "error": "no face detected"
}
```

**Usage:** Add validated face crops to library during bootstrap

---

### Recognize Face

**Endpoint:** `POST /v1/vision/face/recognize`

**Request:**
```json
{
  "image": "<base64_encoded_jpeg>",
  "min_confidence": 0.75
}
```

**Response (Match):**
```json
{
  "success": true,
  "predictions": [
    {
      "userid": "agent_123",
      "confidence": 0.87
    }
  ]
}
```

**Response (No Match):**
```json
{
  "success": true,
  "predictions": []
}
```

**Response (Error):**
```json
{
  "success": false,
  "error": "no face detected"
}
```

**Usage:** Called by IdentityResolver during cross-camera merge attempts

**Threshold:** ≥ 0.75 for auto-merge (configurable in `marengo.yaml`)

---

### Error Handling

**Timeout:** 5 seconds (configurable)

**Retry Logic:**
- Max 3 attempts
- Exponential backoff: 1s, 2s, 4s
- After 3 failures: mark track uncertain, queue for review

**Common Errors:**
- `no face detected` - Face too small, occluded, or profile view
- `connection refused` - CodeProject.AI not running
- `timeout` - Service overloaded

---

## LLM - Review Adjudication

**Service:** Anthropic Claude (configurable provider)

**Model:** `claude-sonnet-4.5` (configurable)

**API Key:** Stored in `secrets.yaml`

---

### Review Track Merge Decision

**Endpoint:** `POST https://api.anthropic.com/v1/messages`

**Request:**
```json
{
  "model": "claude-sonnet-4.5",
  "max_tokens": 1000,
  "temperature": 0.0,
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "<base64_track_A_crop>"
          }
        },
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "<base64_track_B_crop>"
          }
        },
        {
          "type": "text",
          "text": "Are these the same person? Consider clothing, body shape, gait.\n\nContext:\n- Track A: Camera Driveway, exited at portal 'garage_door'\n- Track B: Camera Backyard, entered 4.2s later\n- Grid learning: typical travel time 3.8s\n- Other agents visible: Agent_C (different location)\n\nRespond in JSON format:\n{\n  \"decision\": \"merge\" | \"reject\" | \"uncertain\",\n  \"confidence\": \"high\" | \"medium\" | \"low\",\n  \"reasoning\": \"brief explanation\"\n}"
        }
      ]
    }
  ]
}
```

**Response (Success):**
```json
{
  "id": "msg_123",
  "model": "claude-sonnet-4.5",
  "content": [
    {
      "type": "text",
      "text": "{\n  \"decision\": \"merge\",\n  \"confidence\": \"high\",\n  \"reasoning\": \"Same clothing (blue shirt, dark pants), similar body build, plausible timing.\"\n}"
    }
  ],
  "usage": {
    "input_tokens": 1523,
    "output_tokens": 45
  }
}
```

**Parsing:**
- Extract JSON from `content[0].text`
- Validate structure
- Apply decision via EvidenceProcessor

**Decisions:**
- `merge` + `high` → resolve track
- `reject` + `high` → remove from identity_candidates
- `uncertain` or `low` → leave in queue for human

---

### Error Handling

**Timeout:** 30 seconds (LLM inference can be slow)

**Retry Logic:**
- Max 2 attempts
- 5s delay between attempts
- After 2 failures: leave in queue for human review

**Rate Limiting:**
- Anthropic: 50 requests/minute (tier 2)
- If rate limited: exponential backoff, queue requests

**Common Errors:**
- `overloaded_error` - Retry after 10s
- `invalid_request_error` - Log error, skip LLM for this track
- `authentication_error` - Critical alert, LLM disabled until fixed

---

## Batch Processing

**Face Recognition:** NEEDS SPEC
- Should we batch multiple face recognition calls?
- Currently: one request per track

**LLM:** NEEDS SPEC
- Should we batch multiple review decisions in one prompt?
- Trade-off: cost savings vs. latency
- Currently: one request per uncertain track (immediate processing)

**Decision needed:** Batch optimization worth the complexity?

---

## Cost Tracking

**LLM Usage:**
- Log input/output tokens per request
- Track daily cost (Claude pricing: $3/million input tokens, $15/million output)
- Alert if daily cost > threshold (configurable)

**CodeProject.AI:**
- Free (self-hosted)
- Track request count for performance monitoring only

---

## Configuration

**From `marengo.yaml`:**

```yaml
external_ai:
  face_recognition:
    enabled: true
    api_url: !secret codeproject_url
    timeout: 5
    retry_attempts: 3
    min_confidence: 0.75

  llm:
    enabled: true
    provider: "anthropic"
    model: "claude-sonnet-4.5"
    api_key: !secret llm_api_key
    max_tokens: 1000
    temperature: 0.0
    timeout: 30
    retry_attempts: 2
```

---

## Notes

- **Face recognition:** Synchronous (blocks until response or timeout)
- **LLM:** Synchronous but slow (30s timeout)
- **No batching:** Process evidence immediately as it arrives
- **Fallback:** On failure, queue for human review (system degrades gracefully)
