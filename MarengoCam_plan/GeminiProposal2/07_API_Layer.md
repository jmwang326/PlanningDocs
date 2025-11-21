# 07 - API Layer (The Connectors)

## Purpose
Defines the REST API that serves as the interface between the Backend (Logic/DB) and the Frontend (Dashboard).

**Philosophy:**
- **RESTful:** Standard resource-oriented URLs.
- **Thin Layer:** The API does minimal logic; it mostly queries the DB or calls Component functions.
- **Type-Safe:** Uses Pydantic models for validation.

---

## 1. Interface Contract

### 1.1. Tracks & Entities (Read)
```http
GET /tracks
  Query: start_time, end_time, camera_id, global_entity_id
  Response: List[LocalTrack]

GET /entities
  Query: min_confidence, is_confirmed
  Response: List[GlobalEntity]

GET /entities/{id}/timeline
  Response: List[LocalTrack] (Ordered by time)
```

### 1.2. Review & Feedback (Write)
```http
POST /decisions/merge
  Body: { "source_track_id": "...", "target_entity_id": "..." }
  Action: 
    1. Update DB (reassign ID).
    2. Trigger IdentityResolver.learn_from_merge().
    3. Return Success.

POST /decisions/split
  Body: { "track_id": "...", "new_entity_label": "..." }
  Action: Create new GlobalEntity, move track to it.
```

### 1.3. System & Config
```http
GET /system/health
  Response: { "cpu": 20, "gpu": 50, "queue": 5, "alerts": [] }

GET /config
  Response: Full JSON config.

POST /config
  Body: { "key": "camera.front.fps", "value": 15 }
  Action: Update DB, Trigger Hot Reload.
```

---

## 2. Implementation Details
- **Framework:** FastAPI (Python).
- **Docs:** Auto-generated Swagger UI at `/docs`.
- **Auth:** Basic Auth (Phase 1), JWT (Phase 2).
