# 09 - DevTools & Ops (The Lab)

## Purpose
Tools for development, debugging, simulation, and deployment. This ensures the system is testable and reproducible.

---

## 1. The Replay Harness ("The Time Machine")

### 1.1. Concept
Allows developers to re-run the `ChunkProcessor` and `IdentityResolver` on *past* video data using *current* code.

### 1.2. Usage
```bash
# Re-process yesterday's footage with the new Identity Algorithm
python dev_tools/replay.py \
  --camera front_door \
  --start "2023-10-27 08:00" \
  --end "2023-10-27 09:00" \
  --dry-run  # Don't write to DB, just print decisions
```

### 1.3. Mechanism
1.  Reads `local_tracks` from DB (to get original timestamps).
2.  Locates raw video chunks on disk.
3.  Feeds video into `ChunkProcessor` pipeline.
4.  Compares new output vs old DB records.

---

## 2. The Grid Simulator

### 2.1. Concept
Test the `IdentityResolver` logic without needing video files.

### 2.2. Usage
```bash
# Simulate a person walking A -> B -> C
python dev_tools/simulate_grid.py --scenario "normal_walk"
```

### 2.3. Scenarios (Stress Tests)
- **Normal Walk:** A -> B in 5s. (Expect: Merge).
- **Teleport:** A -> B in 0.1s. (Expect: Split).
- **Doppelgänger:** A and B active simultaneously. (Expect: Split).

---

## 3. Deployment (Infrastructure)

### 3.1. Docker Compose
Defines the runtime environment.
```yaml
services:
  db:
    image: postgres:15
    volumes: ["./data/db:/var/lib/postgresql/data"]
  
  ingest:
    build: ./src/ingest
    deploy:
      resources:
        reservations:
          devices: [{driver: nvidia, count: 1, capabilities: [gpu]}]
    environment:
      - VRAM_LIMIT=3GB
  
  web:
    build: ./src/web
    ports: ["8000:8000"]
```

### 3.2. Database Migrations
- **Tool:** Alembic (Python).
- **Policy:** All schema changes must have a migration script.
