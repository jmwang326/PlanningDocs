# L3T: Configuration Technical Details

This document provides the specific YAML structure and validation rules for the `marengo.yaml` configuration file.

---

## Root Structure

```yaml
system: { ... }
cameras: [ ... ]
portals: [ ... ]
thresholds: { ... }
alerts: { ... }
external_ai: { ... }
storage: { ... }
database: { ... }
stage: "PROD_LEARNING" # or "PROD_AUTO", "DEV_FULL", "DEV_SLOW"
```

---

## `cameras` Section

Defines the properties for each camera.

```yaml
cameras:
  - id: "Driveway"
    enabled: true
    streams:
      main:
        url: !secret driveway_main_url
        resolution: "1920x1080"
      sub:
        url: !secret driveway_sub_url
        resolution: "640x480"
    grid:
      width: 6
      height: 6
    # Per-camera threshold overrides can be placed here
```

---

## `portals` Section

Defines fixed portals where agents can appear or disappear.

```yaml
portals:
  - id: "garage_door"
    camera_id: "Driveway"
    cells: [[3, 2], [3, 3]] # List of [row, col] cells
    type: "structure_entry"
```

---

## `thresholds` Section

Contains all numerical thresholds that govern system decisions.

```yaml
thresholds:
  detection:
    person: 0.5
    vehicle: 0.6
  identity_resolution:
    face_match_threshold: 0.75
    overlap_zone_seconds: 0.4
    grid_timing_variance_percent: 0.3
  review_queue:
    recency_weight: 0.4
    simplicity_weight: 0.3
    impact_weight: 0.3
```

---

## `external_ai` Section

Configures connections to external AI services like CodeProject.AI and LLMs.

```yaml
external_ai:
  face_recognition:
    enabled: true
    api_url: !secret codeproject_url
    timeout: 5
  llm:
    enabled: true
    provider: "anthropic"
    model: "claude-3-sonnet-20240229"
    api_key: !secret llm_api_key
```

---

## `database` and `storage` Sections

Define connection parameters for the database and paths for file storage.

```yaml
database:
  type: "postgresql"
  host: "localhost"
  port: 5432
  database: "marengocam"
  username: !secret db_username
  password: !secret db_password

storage:
  frames:
    base_path: "/data/frames"
    retention_days: 30
```

---

## Configuration Validation and Schema

To ensure robustness and prevent errors from typos or incorrect data types, the `marengo.yaml` file **must** be validated against a strict schema on system startup.

-   **Technology:** The validation shall be implemented using **Pydantic**. A series of Pydantic models will be defined in the Python codebase to mirror the structure of the YAML file.
-   **On Startup:** The system will load the YAML file and parse it directly into the root Pydantic model.
-   **Failure Mode:** If validation fails for any reason (misspelled keys, incorrect data types, values out of range), Pydantic will raise a detailed `ValidationError`. The system **must** catch this exception, print a user-friendly error message indicating the exact location of the problem in the YAML file, and immediately exit. The system will refuse to start with an invalid configuration.

This approach transforms configuration from a potential source of subtle bugs into a clear, validated, and type-safe contract.

---

## Secrets Management

- **Mechanism:** A `!secret` YAML tag is used to reference sensitive values.
- **Storage:** Secrets are stored in a separate, encrypted `secrets.yaml` file.
- **Access:** A CLI (`marengocam secrets ...`) is provided to securely set, get, and list secrets.
- **Example:** `url: !secret driveway_main_url`
