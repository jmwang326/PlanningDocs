# Level 3 - GUI & Presentation Layer

## Purpose
Tactical guide for the MarengoCam graphical user interface. Defines the UI strategy, application structure, and user workflows for reviewing, managing, and monitoring the system.

**See also:** [L2_DecisionArchitecture.md](L2_DecisionArchitecture.md) for GUI component overview.

---

## UI Strategy

### Separation of Concerns

**Philosophy:** Build separate, focused applications rather than a monolithic dashboard.

**Why:**
- Focused development (one app per phase)
- Progressive refinement (iterate independently)
- Simpler deployment (only deploy what's needed)
- Easier maintenance (isolated concerns)

### Technology Choices

**Web UIs (Primary):**
- Framework: Flask/FastAPI + React/Vue
- Use for: Review queue, timeline viewer, health dashboard
- Advantages: Cross-platform, no installation, mobile-friendly

**Desktop Tools (Secondary):**
- Framework: tkinter
- Use for: Initial setup, grid visualization, low-level debugging
- Advantages: Fast iteration, direct file access, simple deployment

**CLI Tools (Administrative):**
- Framework: Click/Typer
- Use for: Configuration, health checks, automation
- Advantages: Scriptable, lightweight, no UI dependencies

---

## Primary Applications

### 1. Review Queue Interface

**Purpose:** Human/LLM adjudication of uncertain tracks

**Key workflows:**
1. View prioritized queue of uncertain tracks
2. Open review panel for selected track
3. Make merge/reject/skip decision
4. Provide optional reasoning

**Specifications:**
- **L4_ReviewQueue.md** - Queue display, priority, filtering
- **L4_ReviewPanel.md** - Identity comparison panel

**Target users:**
- Humans (bootstrap phase, high-priority reviews)
- LLM (automated review, batch processing)

**Access method:** Web UI (primary) + API endpoint (for LLM)

---

### 2. Timeline Viewer

**Purpose:** Browse, filter, and replay completed agent timelines

**Key workflows:**
1. Select agent (by name or ID)
2. View chronological timeline of tracks
3. Filter by camera, time range, confidence
4. Replay video clips in sequence
5. Jump to specific track or event

**Specifications:**
- **L4_Timeline.md** - Timeline display, filtering, playback

**Target users:** End users (family members, administrators)

**Access method:** Web UI

**Status:** Planned for Phase 2 (after bootstrap)

---

### 3. System Health Dashboard

**Purpose:** Monitor system performance, detect issues, view alerts

**Key workflows:**
1. View real-time processing status
2. Monitor queue depths and auto-merge rate
3. Investigate alerts and warnings
4. Drill down to per-camera performance
5. View historical trends

**Specifications:**
- **L4_HealthDashboard.md** - Dashboard layout, metrics, alerts

**Target users:** System administrators, developers

**Access method:** Web UI (primary) + CLI (status checks)

---

### 4. Configuration Tool

**Purpose:** Configure cameras, portals, thresholds, alerts

**Key workflows:**
1. Add/edit/delete cameras
2. Configure portals (garage doors, entries)
3. Set detection thresholds (confidence, duration)
4. Configure alert channels (email, log)
5. Manage grid learning (view learned paths)

**Specifications:**
- **L4_ConfigurationUI.md** - Configuration interface

**Target users:** System administrators

**Access method:** Web UI (primary) + config files (advanced)

**See also:** [L3_Configuration.md](L3_Configuration.md) for configuration data model

---

## Secondary Applications

### 5. Face Library Manager

**Purpose:** Review and manage face recognition training data

**Key workflows:**
1. View all face crops per agent
2. Add/remove training samples
3. Correct misassigned faces
4. Retrain face recognition model

**Specifications:**
- **L4_FaceLibrary.md** (planned)

**Target users:** System administrators

**Access method:** Web UI

**Status:** Planned for Phase 3 (after face library is populated)

---

### 6. Grid Visualization Tool

**Purpose:** Visualize learned grid paths between cameras

**Key workflows:**
1. Select camera pair (Camera_A → Camera_B)
2. View 6×6 grid heatmap (travel times)
3. Identify overlap zones (< 0.4s)
4. Debug travel time anomalies

**Specifications:**
- **L4_GridVisualization.md** (planned)

**Target users:** Developers, system administrators

**Access method:** Desktop UI (tkinter)

**Status:** Planned for Phase 2 (debugging tool)

---

### 7. CLI Health Monitor

**Purpose:** Command-line system diagnostics and health checks

**Key commands:**
```bash
# View system status
marengocam status

# Check processing lag
marengocam health --metric lag

# View queue depths
marengocam queues

# Export metrics
marengocam export --output metrics.csv

# Restart components
marengocam restart inference-manager
```

**Target users:** Developers, automation scripts

**Access method:** CLI

**Status:** Planned for Phase 1 (essential for debugging)

---

## UI Architecture

### Component Layers

**Presentation Layer (Frontend):**
- React/Vue components
- WebSocket for real-time updates
- REST API client

**API Layer (Backend):**
- Flask/FastAPI endpoints
- Authentication/authorization
- Request validation

**Business Logic Layer:**
- Handlers (ChunkProcessor, TemporalLinker, etc.)
- Database queries
- Event processing

**Data Layer:**
- SQLAlchemy ORM
- PostgreSQL database
- File storage (video frames)

---

### API Endpoints

**Review Queue:**
- `GET /api/queue` - Fetch prioritized queue
- `POST /api/queue/{track_id}/review` - Submit review decision
- `GET /api/queue/{track_id}/panel` - Get review panel data

**Timeline:**
- `GET /api/agents` - List all agents
- `GET /api/agents/{agent_id}/timeline` - Fetch agent timeline
- `GET /api/tracks/{track_id}` - Get track details
- `GET /api/tracks/{track_id}/video` - Stream track video

**Health:**
- `GET /api/health` - Get current system health
- `GET /api/health/metrics` - Get historical metrics
- `GET /api/health/alerts` - Get active alerts
- `POST /api/health/alerts/{alert_id}/ack` - Acknowledge alert

**Configuration:**
- `GET /api/config/cameras` - List cameras
- `POST /api/config/cameras` - Add/update camera
- `GET /api/config/portals` - List portals
- `POST /api/config/portals` - Add/update portal

**See:** L14_APIServer.py for implementation

---

## Authentication & Access Control

### User Roles

**Admin:**
- Full access (all applications)
- Can modify configuration
- Can manage face library

**Reviewer:**
- Review queue access
- Cannot modify configuration
- Cannot manage face library

**Viewer:**
- Timeline viewer access only
- Read-only

**Developer:**
- Admin + CLI access
- Debug tools access

### Authentication Methods

**Web UI:**
- Session-based (login page)
- JWT tokens for API access

**CLI:**
- API key (stored in config file)
- Environment variable (CI/CD)

---

## UI/UX Guidelines

### Design Principles

**1. Clarity over aesthetics:**
- Functional UI (no unnecessary decoration)
- Clear labels and tooltips
- Consistent layout

**2. Context over navigation:**
- Show relevant information inline
- Minimize page transitions
- Drill-down from summary to detail

**3. Feedback over silence:**
- Show processing state (loading indicators)
- Confirm actions (success/error messages)
- Real-time updates (WebSocket)

**4. Defaults over configuration:**
- Sensible defaults (minimal initial config)
- Progressive disclosure (advanced options hidden)
- Auto-detection where possible

---

### Visual Style

**Color coding:**
- Green: Normal/success
- Yellow: Warning
- Red: Critical/error
- Blue: Info

**Typography:**
- Monospace for IDs, timestamps, metrics
- Sans-serif for text, labels

**Layout:**
- Grid-based (responsive)
- Card-based components
- Consistent spacing

---

## Development Phases

### Phase 1: Bootstrap (Essential)
1. Review Queue Interface ✅ (Priority 1)
2. System Health Dashboard ✅ (Priority 2)
3. CLI Health Monitor (Priority 3)

**Goal:** Get system operational, monitor health, train system

---

### Phase 2: Production (Core Features)
1. Timeline Viewer (Priority 1)
2. Configuration Tool (Priority 2)
3. Grid Visualization Tool (Priority 3)

**Goal:** Make system usable for end users

---

### Phase 3: Refinement (Advanced Features)
1. Face Library Manager (Priority 1)
2. Mobile apps (Priority 2)
3. Advanced analytics (Priority 3)

**Goal:** Improve system accuracy and usability

---

## Related Documents

### Architecture
- **L2_DecisionArchitecture.md** - GUI component overview

### Other L3 Components
- **L3_EvidenceProcessing.md** - Review queue source data
- **L3_SystemHealth.md** - Health dashboard source data
- **L3_Configuration.md** - Configuration data model

### L4 Specifications
- **L4_ReviewQueue.md** - Review queue interface specification
- **L4_ReviewPanel.md** - Identity comparison panel specification
- **L4_HealthDashboard.md** - System health dashboard specification
- **L4_ConfigurationUI.md** (planned) - Configuration tool specification
- **L4_Timeline.md** (planned) - Timeline viewer specification
- **L4_FaceLibrary.md** (planned) - Face library manager specification
- **L4_GridVisualization.md** (planned) - Grid visualization tool specification

### Implementation
- **L14_APIServer.py** - Flask/FastAPI API server
- **L14_ReviewUI.py** - Review queue frontend
- **L14_HealthDashboard.py** - Health dashboard frontend
- **L14_TimelineViewer.py** - Timeline viewer frontend
