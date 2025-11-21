# 05 - Configuration (The Controls)

## Purpose
Defines the configuration schema, validation logic, and the mechanism for dynamic updates ("Hot Reload").

**Philosophy:**
- **Database as Authority:** Runtime settings live in the DB, not YAML.
- **Safe Changes:** Invalid configs are rejected before they are applied.
- **Secrets Isolation:** Passwords/Keys stay in `secrets.yaml` or ENV vars, never in the DB.

---

## 1. Interface Contract

### 1.1. Primary Functions
```python
def get_config(key: str) -> Any:
    """
    Read from Cache -> DB -> Defaults.
    """

def update_config(key: str, value: Any, user: str) -> bool:
    """
    1. Validate value against schema.
    2. Update DB.
    3. Publish 'ConfigChanged' event.
    """
```

### 1.2. Evolutionary Stages

**Stage 1: Static YAML**
- **Logic:** Read `config.yaml` on startup. Restart required to change.
- **Goal:** Get the system running quickly.

**Stage 2: Hybrid**
- **Logic:** Cameras in YAML, Rules in DB.

**Stage 3: Full Dynamic**
- **Logic:** Only DB connection in YAML. All else in DB with Hot Reload.

---

## 2. Configuration Schema (Domain Model)

### 2.1. Bootstrap Config (`config.yaml`)
Minimal settings required to start the app.
```yaml
database:
  url: "postgresql://user:pass@localhost:5432/marengo"
redis:
  url: "redis://localhost:6379/0"
directories:
  media_root: "D:/MarengoData"
```

### 2.2. Runtime Config (DB: `system_settings`)

#### **Cameras**
```json
{
  "id": "front_door",
  "rtsp_url": "rtsp://...",
  "substream_url": "rtsp://...",
  "zones": [
    {"name": "porch", "poly": [[0,0], [100,0], ...]},
    {"name": "street", "poly": ...}
  ],
  "fps_limit": 10
}
```

#### **Grid Logic**
```json
{
  "learning_rate": 0.1,
  "min_samples_for_inference": 10,
  "max_travel_time_sigma": 2.0
}
```

#### **Automation Rules**
```json
[
  {
    "name": "Nightly Patrol",
    "condition": "person_detected AND zone='backyard' AND time > '22:00'",
    "action": "mqtt_publish('alarm/trigger')"
  }
]
```

---

## 3. Validation & Hot Reload

### 3.1. Validation Rules
- **Zones:** Polygons must be closed and non-self-intersecting.
- **Cameras:** RTSP URLs must be reachable (ping check) before saving.
- **Dependencies:** Cannot delete a Zone if it is referenced in an Automation Rule.

### 3.2. Hot Reload Mechanism
1.  User updates "Camera FPS" in Dashboard.
2.  API calls `update_config`.
3.  `update_config` validates and writes to DB.
4.  `update_config` publishes `config_update:camera:front_door` to Redis/EventBus.
5.  `ChunkProcessor` (listening on Bus) receives event.
6.  `ChunkProcessor` restarts the FFmpeg capture thread with new settings.
