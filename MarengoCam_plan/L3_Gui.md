# Level 3 - GUI & Presentation Layer

## Purpose
GUI strategy, application structure, user workflows.

---

## UI Strategy

**Philosophy:** Separate, focused applications (not monolithic dashboard).

**Why:** Focused development, progressive refinement, simpler deployment, easier maintenance.

**Technology:**
- **Web UIs:** Flask/FastAPI + React/Vue (review queue, timeline viewer, health dashboard)
- **Desktop tools:** tkinter (initial setup, grid visualization, debugging)
- **CLI tools:** Click/Typer (configuration, health checks, automation)

---

## Primary Applications

### 1. Review Queue Interface

**Purpose:** Human/LLM adjudication of uncertain tracks

**Key workflows:**
1. View prioritized queue
2. Open review panel for selected track
3. Make merge/reject/skip decision
4. Provide optional reasoning

**Access:** Web UI (humans) + API endpoint (LLM)

**Specs:** L4_ReviewQueue.md, L4_ReviewPanel.md

---

### 2. Timeline Viewer

**Purpose:** Browse, filter, replay agent timelines

**Key workflows:**
1. Select agent (by name or ID)
2. View chronological timeline
3. Filter by camera, time range, confidence
4. Replay video clips in sequence

**Access:** Web UI

**Status:** Phase 2 (after bootstrap)

**Specs:** L4_Timeline.md

---

### 3. System Health Dashboard

**Purpose:** Monitor performance, detect issues, view alerts

**Key workflows:**
1. View real-time processing status
2. Monitor queue depths, auto-merge rate
3. Investigate alerts and warnings
4. Drill down to per-camera performance
5. View historical trends

**Access:** Web UI (primary) + CLI (status checks)

**Specs:** L4_HealthDashboard.md

---

### 4. Configuration Tool

**Purpose:** Configure cameras, portals, thresholds

**Key workflows:**
1. Add/edit/delete cameras
2. Configure portals (doors, gates)
3. Set detection thresholds
4. Manage grid learning

**Access:** Web UI (primary) + config files (advanced)

**Specs:** L4_ConfigurationUI.md

---

## Secondary Applications

### 5. Face Library Manager

**Purpose:** Manage face recognition training data

**Key workflows:**
1. View all face crops per agent
2. Add/remove training samples
3. Correct misassigned faces
4. Retrain face recognition model

**Access:** Web UI

**Status:** Phase 3 (after face library populated)

---

### 6. Grid Visualization Tool

**Purpose:** Visualize learned grid paths between cameras

**Key workflows:**
1. Select camera pair
2. View 6×6 heatmap (travel times)
3. Identify overlap zones (< 0.4s)
4. Debug travel time anomalies

**Access:** Desktop UI (tkinter)

**Status:** Phase 2 (debugging tool)

---

### 7. CLI Health Monitor

**Purpose:** Command-line diagnostics

**Key commands:**
```bash
marengocam status
marengocam health --metric lag
marengocam queues
marengocam export --output metrics.csv
marengocam restart inference-manager
```

**Access:** CLI

**Status:** Phase 1 (essential)

---

## UI Architecture

**Presentation Layer:** React/Vue components, WebSocket, REST API client

**API Layer:** Flask/FastAPI endpoints, auth, validation

**Business Logic:** Handlers, database queries, event processing

**Data Layer:** SQLAlchemy ORM, PostgreSQL, file storage

---

## API Endpoints

**Review Queue:**
- `GET /api/queue` - Fetch prioritized queue
- `POST /api/queue/{track_id}/review` - Submit decision
- `GET /api/queue/{track_id}/panel` - Get review panel

**Timeline:**
- `GET /api/agents` - List agents
- `GET /api/agents/{agent_id}/timeline` - Fetch timeline
- `GET /api/tracks/{track_id}` - Get track details
- `GET /api/tracks/{track_id}/video` - Stream video

**Health:**
- `GET /api/health` - Current status
- `GET /api/health/metrics` - Historical metrics
- `GET /api/health/alerts` - Active alerts

**Configuration:**
- `GET /api/config/cameras` - List cameras
- `POST /api/config/cameras` - Add/update camera
- `GET /api/config/portals` - List portals
- `POST /api/config/portals` - Add/update portal

---

## Development Phases

**Phase 1 (Bootstrap):**
1. Review Queue Interface
2. System Health Dashboard
3. CLI Health Monitor

**Phase 2 (Production):**
1. Timeline Viewer
2. Configuration Tool
3. Grid Visualization Tool

**Phase 3 (Refinement):**
1. Face Library Manager
2. Advanced analytics
