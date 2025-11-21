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

#### **Grid Logic**
```json
{
  "learning_rate": 0.1,
  "min_samples_for_inference": 10,
  "max_travel_time_sigma": 2.0
}
```

#### **LLM/External API**
```json
{
  "endpoint": "http://localhost:5001/v1/chat/completions",
  "api_key_secret": "marengo_llm_api_key",
  "model": "gpt-4-turbo"
}
```

#### **Portals**
```json
[
    {
        "name": "FrontDoor_to_Driveway",
        "from_camera": "Front_Door",
        "to_camera": "Driveway",
        "from_zone": "porch_exit",
        "to_zone": "driveway_entry",
        "coordinates": [[100, 200], [300, 400]]
    }
]
```

---

## 3. Validation & Hot Reload

### 3.1. Validation Rules
- **Grid Logic:** `learning_rate` must be between 0 and 1.
- **Portals:** Coordinates must be valid points.
- **Secrets:** API keys and other secrets are not stored directly in this configuration but are referenced by a key. The system should validate that a secret exists for the given key.

### 3.2. Hot Reload Mechanism
1.  User updates a configuration value in the Dashboard (e.g., `learning_rate`).
2.  API calls `update_config`.
3.  `update_config` validates and writes the new value to the database.
4.  `update_config` publishes a `config_update:<key>` event (e.g., `config_update:grid_logic`) to the event bus.
5.  Relevant components (e.g., `IdentityResolver`) listening on the bus receive the event.
6.  The component reloads its internal state with the new configuration value.
