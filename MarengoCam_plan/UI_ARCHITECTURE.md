# UI Architecture — Separate Apps Per Phase

This document defines the UI strategy for MarengoCam: **separate, simple applications per phase** that focus on specific tasks. Consolidate later only if needed.

---

## Philosophy

**Separation of Concerns:**
- Each UI serves a distinct purpose (config, viewing, training, health monitoring)
- Simple, focused apps easier to debug than monolithic dashboard
- Different users/roles may need different UIs (admin vs. security review)

**Development First:**
- Prove backend logic before building polished UI
- Quick-and-dirty debugging UIs acceptable early
- Progressive refinement as system matures

**Technology Stack:**
- **Desktop viewers:** Python + tkinter (already works, fast iteration, no web server needed)
- **Web dashboards:** Flask/FastAPI + simple HTML/JS (lightweight, no framework complexity)
- **CLI tools:** Click/Typer for admin tasks (health, config, secrets)

**Consolidation Strategy:**
- Keep separate apps through Phase 3
- Phase 4+: Consider unified web UI if workflow proven valuable
- Never consolidate if separation works better

---

## UI Apps by Phase

### **Phase 0: Development & Debugging**

#### 1. **Camera Grid Viewer** (Desktop - tkinter)
**Status:** ✅ Already exists (`sandbox_cameraconfig/camera_dashboard_4x5.py`)

**Purpose:**
- View live Blue Iris camera feeds in 4×5 grid
- Toggle 32×32 grid overlay (press `g`)
- Draw ignore zones (press `a` for AOI edit mode)
- Verify camera connectivity before starting pipeline

**Features:**
- Fetches JPEG snapshots from BI `/image/{short}` endpoint
- 32×32 uniform grid overlay (updated from old 12×12)
- Polygon drawing for ignore zones (normalized coordinates)
- Saves ignore zones to `overlay_config.json`

**Users:** Developer (initial setup, camera verification)

**Tech:** Python + tkinter + PIL + requests

**Run:**
```powershell
python sandbox_cameraconfig/camera_dashboard_4x5.py
```

**Updates Needed:**
- Update grid overlay to 32×32 uniform (currently 12×12 perspective)
- Remove perspective parameters (k_row, horiz_vanish, etc.)
- Keep ignore zone polygon drawing

---

#### 2. **CLI Health Monitor** (Terminal - Click)
**Status:** 🔨 To build (Phase 0)

**Purpose:**
- Check system health without web UI
- Quick diagnostics during development
- Config management commands

**Commands:**
```powershell
# Health checks
marengo health status                    # Overall system status
marengo health cameras                   # Per-camera frame rates, detections
marengo health services                  # External AI service health
marengo health storage                   # Disk usage, retention stats

# Configuration
marengo config show                      # Display current config
marengo config validate                  # Test config before applying
marengo config reload                    # Hot-reload config
marengo config rollback --version <sha>  # Rollback to previous config

# Secrets management
marengo secrets set bi_password          # Encrypt and store secret
marengo secrets reveal bi_password --timeout 30s
marengo secrets test blue_iris           # Validate BI credentials

# System control
marengo start --stage DEV_SLOW           # Start with specific stage
marengo stop                             # Graceful shutdown
marengo restart --stage DEV_FULL         # Change stage and restart
```

**Users:** Developer, admin

**Tech:** Python Click + Rich (terminal formatting)

**Priority:** Phase 0 (build alongside core pipeline)

---

### **Phase 1: Intra-Camera Tracking**

#### 3. **Single-Camera Debug Viewer** (Desktop - tkinter)
**Status:** 🔨 To build (Phase 1)

**Purpose:**
- Watch one camera in detail
- Visualize detection bboxes, track IDs, cell assignments
- Debug ID swaps, height calibration, occlusions

**Features:**
- Select camera from dropdown
- Show current frame with overlays:
  - Detection bboxes (color-coded by class: green=person, blue=vehicle, yellow=animal)
  - Track ID labels above bbox
  - 32×32 grid overlay (toggle with `g`)
  - Cell ID under cursor (hover)
  - Current cell highlighted for each track
- Right panel: Track list with stats
  - Track ID, class, duration, entry/exit cells
  - Quality score, detection count
  - Height calibration status per cell (bar chart)
- Playback controls: pause, step frame, speed slider
- Camera state indicator: Standby/Armed/Active/Post

**Users:** Developer (debugging tracking logic)

**Tech:** Python + tkinter + OpenCV (frame decoding)

**Run:**
```powershell
python tools/single_camera_viewer.py --camera FrontGate
```

**Priority:** Phase 1 (after basic tracking works)

---

### **Phase 2: Cross-Camera Merging**

#### 4. **Merge Decision Queue** (Web - Flask + Simple HTML)
**Status:** 🔨 To build (Phase 2)

**Purpose:**
- Review medium-confidence merge candidates (0.50-0.80 score)
- Accept/reject/defer merge decisions with **identity naming**
- Provide ground truth for portal timing and face library learning

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Merge Queue (23 pending)                            │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Candidate #1 (confidence: 0.68)                   │
│                                                     │
│  ┌──────────────┐     ┌──────────────┐            │
│  │  Camera A    │  →  │  Camera B    │            │
│  │  Agent #42   │     │  Agent #89   │            │
│  │  12:34:15 PM │     │  12:34:18 PM │            │
│  │              │     │              │            │
│  │  [Panel: 2x6]│     │  [Panel: 2x6]│            │
│  │   images     │     │   images     │            │
│  └──────────────┘     └──────────────┘            │
│                                                     │
│  Evidence:                                         │
│    • Δt: 3.2s (plausible portal crossing)         │
│    • Portal: FrontDoor → Driveway (matched)       │
│    • Face match: 0.72 (medium confidence)         │
│    • Observable: similar clothing, build          │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │ Identity Name (required for Accept):        │  │
│  │ ┌────────────────────────────────┐ [New...] │  │
│  │ │ John Doe                    ▼ │          │  │
│  │ └────────────────────────────────┘          │  │
│  │ Recent: John Doe (47×), Jane Smith (23×)   │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  [✓ Accept & Name]  [✗ Reject]  [→ Defer]         │
│  [👁 View Full Track]                              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Features:**
- Queue sorted by confidence (lowest first = most uncertain)
- Side-by-side panel comparison (768×768 each, 2×6 layout)
- Evidence summary with scores
- **Identity picker with autocomplete** (prevents duplicate names)
  - Dropdown shows recent/common identities (sorted by last seen)
  - Search box with fuzzy match warnings ("Jon Doe" → "Did you mean John Doe?")
  - Name normalization (case-insensitive, whitespace-trimmed)
- Accept → links to identity, adds faces to library, updates portal timing stats
- Reject → record as negative example
- Defer → keep in queue for later
- View full track → open track detail viewer

**Users:** Developer, security reviewer (during learning phase)

**Tech:** Flask + Jinja2 templates + minimal CSS/JS

**API Endpoints:**
```python
GET  /merge/queue                    # List pending merges
GET  /merge/candidate/{id}           # Get merge candidate details
POST /merge/candidate/{id}/accept    # Accept merge (body: {identity_name: "..."})
POST /merge/candidate/{id}/reject    # Reject merge
POST /merge/candidate/{id}/defer     # Defer decision
GET  /merge/stats                    # Queue stats, auto-merge rate
GET  /identities/search?q=john       # Search identities for autocomplete
GET  /identities/recent?type=person  # Recent identities for dropdown
```

**Run:**
```powershell
python -m marengo.ui.merge_queue --port 5001
# Open http://localhost:5001
```

**Priority:** Phase 2 (after cross-camera merging implemented)

---

#### 5. **Edge Visualization Dashboard** (Web - Flask + Chart.js)
**Status:** 🔨 To build (Phase 3)

**Purpose:**
- Visualize learned 8×8 inter-camera edges
- Track edge confidence growth over time
- Identify camera pairs with poor learning (no edges)

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Edge Learning Dashboard                             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Camera Pair: [FrontGate ▼] → [DrivewayEast ▼]    │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │  8×8 Heatmap (Edge Confidence)                │ │
│  │                                                │ │
│  │    0  1  2  3  4  5  6  7                     │ │
│  │  0 ░░░░░░░░░░░░░░░░░░░░░░░░                   │ │
│  │  1 ░░██████░░░░░░░░░░░░░░░░                   │ │
│  │  2 ░░████████░░░░░░░░░░░░░░                   │ │
│  │  3 ░░░░████████████░░░░░░░░                   │ │
│  │  4 ░░░░░░████████████░░░░░░                   │ │
│  │  5 ░░░░░░░░████████████░░░░                   │ │
│  │  6 ░░░░░░░░░░████████████░░                   │ │
│  │  7 ░░░░░░░░░░░░░░████████░░                   │ │
│  │                                                │ │
│  │  Click cell for edge details                  │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  Selected Edge: (1,2) → (6,3)                      │
│    Min time: 2.1s                                  │
│    Typical time: 2.8s (P90: 3.4s)                 │
│    Confidence: 0.92 (47 observations)             │
│    Last seen: 5 minutes ago                        │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │  Edge Count Growth (All Camera Pairs)         │ │
│  │                                                │ │
│  │  [Line chart: edges over time]                │ │
│  │   Week 1: 12 edges                             │ │
│  │   Week 2: 45 edges                             │ │
│  │   Week 3: 89 edges                             │ │
│  │   Week 4: 134 edges (current)                  │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  [Export Grid Snapshot]  [Compare Week-Over-Week] │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Features:**
- Dropdown to select camera pair
- 8×8 heatmap: color intensity = edge confidence (0=gray, 1=green)
- Click cell → show edge details (min/typical/P90 times, support count)
- Time-series chart: edge count growth over weeks
- Week-over-week diff view (show new/lost/changed edges)
- Export grid snapshot (JSON download)

**Users:** Developer, admin (monitoring learning progress)

**Tech:** Flask + Chart.js + simple CSS grid

**Run:**
```powershell
python -m marengo.ui.edge_dashboard --port 5002
# Open http://localhost:5002
```

**Priority:** Phase 3 (after edge learning proven)

---

### **Phase 3: Face Recognition & Learning**

#### 6. **Face Library Manager** (Web - Flask)
**Status:** 🔨 To build (Phase 3)

**Purpose:**
- View registered faces in CodeProject.AI library
- Manually add/remove faces
- Correct misidentifications (rename identities)

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Face Library (12 known people)                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [+ Register New Person]         [Search: ___]     │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ Person #1   │  │ Person #2   │  │ Person #3  │ │
│  │ "John"      │  │ "Jane"      │  │ "Unknown1" │ │
│  │             │  │             │  │            │ │
│  │ [Face 1/5]  │  │ [Face 1/3]  │  │ [Face 1/2] │ │
│  │  📷          │  │  📷          │  │  📷         │ │
│  │             │  │             │  │            │ │
│  │ 47 matches  │  │ 23 matches  │  │ 2 matches  │ │
│  │ Last: 1h ago│  │ Last: 3d ago│  │ Last: now  │ │
│  │             │  │             │  │            │ │
│  │ [Edit]      │  │ [Edit]      │  │ [Edit]     │ │
│  └─────────────┘  └─────────────┘  └────────────┘ │
│                                                     │
└─────────────────────────────────────────────────────┘

Click [Edit] on Person #3:
┌─────────────────────────────────────────────────────┐
│ Edit Person: "Unknown1"                             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Name: [Unknown1_______]  [Rename to: _______]     │
│                                                     │
│  Registered Faces (2):                             │
│  ┌──────┐  ┌──────┐                                │
│  │ Face1│  │ Face2│                                │
│  │ 📷   │  │ 📷   │                                │
│  │0.82  │  │0.76  │  (quality scores)             │
│  │[✗Del]│  │[✗Del]│                                │
│  └──────┘  └──────┘                                │
│                                                     │
│  Recent Matches (last 10):                         │
│    • 2025-11-07 14:32 - FrontGate (conf: 0.84)    │
│    • 2025-11-07 12:15 - BackYard (conf: 0.79)     │
│    ...                                             │
│                                                     │
│  [Add Face from Track]  [Merge with Other Person] │
│  [Delete Person]                                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Features:**
- Gallery view of all registered people
- Face count per person (CodeProject.AI supports multiple faces)
- Match statistics (count, last seen)
- Rename identities (unknown → known name)
- Delete faces (remove poor quality)
- Merge people (combine duplicates)
- Add face from track (select track, choose best frame)

**Users:** Security reviewer, admin

**Tech:** Flask + simple grid layout

**Run:**
```powershell
python -m marengo.ui.face_library --port 5003
# Open http://localhost:5003
```

**Priority:** Phase 3 (after face recognition integrated)

---

### **Phase 4+: Timeline & Review**

#### 7. **Event Timeline Viewer** (Web - Flask)
**Status:** 🔨 To build (Phase 4)

**Purpose:**
- Browse detected activity chronologically
- Filter by agent, camera, time range
- Reconstruct video clips from JPEGs

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│ Activity Timeline - November 7, 2025                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Filters:                                          │
│   Date: [2025-11-07 ▼]  Person: [All ▼]          │
│   Camera: [All Cameras ▼]  Class: [All ▼]        │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │ 14:32:15 - Person #42 (John)                │  │
│  │ ─────────────────────────────────────────────│  │
│  │ FrontGate → DrivewayEast → BackYard          │  │
│  │ Duration: 2m 34s                              │  │
│  │ [▶ Play] [📷 Thumbnails] [📊 Details]        │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │ 12:15:42 - Vehicle #89                       │  │
│  │ ─────────────────────────────────────────────│  │
│  │ DrivewayEast → Garage                        │  │
│  │ Plate: ABC-1234 (conf: 0.88)                 │  │
│  │ Duration: 45s                                 │  │
│  │ [▶ Play] [📷 Thumbnails] [📊 Details]        │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │ 08:23:11 - Person #12 (Jane)                │  │
│  │ ─────────────────────────────────────────────│  │
│  │ BackYard only (no cross-camera)              │  │
│  │ Duration: 5m 12s                              │  │
│  │ [▶ Play] [📷 Thumbnails] [📊 Details]        │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
└─────────────────────────────────────────────────────┘

Click [▶ Play]:
┌─────────────────────────────────────────────────────┐
│ Playback: Person #42 (John) - 14:32:15             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │                                              │  │
│  │        [Video reconstructed from JPEGs]      │  │
│  │                                              │  │
│  │        Camera: FrontGate → DrivewayEast      │  │
│  │        00:23 / 02:34                         │  │
│  │                                              │  │
│  │  [◀◀] [▶/❚❚] [▶▶]  ────────●────────        │  │
│  │                                              │  │
│  │  Speed: [0.5x] [1x] [2x] [4x]               │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  Camera transitions:                               │
│    • 14:32:15 - FrontGate (entry)                 │
│    • 14:32:18 - DrivewayEast (merge: 0.92)        │
│    • 14:33:42 - BackYard (merge: 0.88)            │
│    • 14:34:49 - (exit structure)                  │
│                                                     │
│  [📥 Download MP4]  [🔗 Share Link]                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Features:**
- Chronological list of activity (agent-centric view)
- Filter by date, person, camera, class
- Play button → reconstruct MP4 from JPEG manifests (10 FPS playback)
- Multi-camera playback: automatically switches cameras at merge points
- Download MP4 for external review
- Thumbnail gallery view (grid of key frames)
- Details view: show all tracks, merge decisions, evidence scores

**Users:** Security reviewer, homeowner

**Tech:** Flask + Video.js (HTML5 video player) + ffmpeg (JPEG → MP4)

**Run:**
```powershell
python -m marengo.ui.timeline --port 5004
# Open http://localhost:5004
```

**Priority:** Phase 4 (after merging stable)

---

### **Phase 5: Advanced (Optional)**

#### 8. **Configuration Studio** (Web - Flask + Drag-Drop UI)
**Status:** 🔨 Deferred (Phase 5)

**Purpose:**
- Drag cameras to declare adjacency hints
- Set per-camera flags (vehicle_eligible, use_lpr)
- Mark AOIs (Door, Garage, Exit) on 32×32 grid
- Draw ignore zones

**Features:**
- Drag-and-drop camera tiles to define adjacency
- Checkbox toggles for flags
- Grid overlay on camera thumbnail for AOI placement
- Polygon drawing for ignore zones
- Session/undo/commit pattern

**Users:** Admin (initial setup, occasional reconfiguration)

**Tech:** Flask + JavaScript drag-drop library

**Priority:** Phase 5 (system works without this; nice-to-have for complex setups)

**Note:** Phase 2-3 UIs (Merge Queue + Face Library Manager) already provide manual review/training capabilities. Configuration Studio is optional enhancement for large deployments.

---

## Alternative: "Learning Studio" (Unified Training UI)

**Status:** 🤔 Concept for Phase 5+ (alternative to separate Merge Queue + Face Library)

If separate UIs (Merge Queue, Face Library) feel disconnected, consider a unified "Learning Studio" that combines:

**Boards:**
1. **Unmatched Faces** - Unknown face crops awaiting identity assignment
2. **Identities** - Registered people with exemplars (drag faces to assign)
3. **Merge Candidates** - Cross-camera continuations (CCC) to accept/reject

**Workflow:**
- User sees all learning tasks in one UI (faces + merges)
- Drag face crop to identity → registers to CodeProject.AI library
- Accept merge → updates edges, registers face if new
- Single session/undo/commit for all changes

**Pros:** Unified workflow, less context switching  
**Cons:** More complex UI, harder to build/maintain

**Decision:** Start with separate UIs (Phase 2-3); reconsider unified studio only if workflow problems emerge.

---

## UI Architecture Summary

| Phase | UI App | Type | Port | Purpose | Status |
|-------|--------|------|------|---------|--------|
| **0** | Camera Grid Viewer | Desktop (tkinter) | N/A | Live camera feeds, grid overlay, ignore zones | ✅ Exists (needs 32×32 update) |
| **0** | CLI Health Monitor | Terminal (Click) | N/A | System health, config management, secrets | 🔨 Build |
| **1** | Single-Camera Debug Viewer | Desktop (tkinter) | N/A | Track visualization, cell assignment, height calibration | 🔨 Build |
| **2** | Merge Decision Queue | Web (Flask) | 5001 | Review merge candidates, provide ground truth | 🔨 Build |
| **3** | Edge Visualization Dashboard | Web (Flask) | 5002 | Visualize learned edges, confidence growth | 🔨 Build |
| **3** | Face Library Manager | Web (Flask) | 5003 | Manage registered faces, correct identities | 🔨 Build |
| **4** | Event Timeline Viewer | Web (Flask) | 5004 | Browse activity, play reconstructed clips | 🔨 Build |
| **5** | Configuration Studio | Web (Flask) | 5005 | Adjacency hints, AOI placement, camera flags | 🔨 Deferred |

---

## Technology Decisions

### Desktop Apps (tkinter)
**Pros:**
- Fast iteration (no web server, no HTML/CSS/JS)
- Direct filesystem access (read configs, frames)
- Good for development/debugging tools
- Native feel on Windows

**Cons:**
- Not accessible remotely
- Limited to developer machine
- Not suitable for production review workflow

**Use for:** Phase 0-1 development tools, camera setup

---

### Web Apps (Flask + Simple HTML)
**Pros:**
- Accessible from any device (phone, tablet, desktop)
- Share links for remote review
- Multiple users can access simultaneously
- Standard browser-based UX

**Cons:**
- Requires web server process
- More complex than tkinter (HTML/CSS/JS)
- Security considerations (auth, HTTPS)

**Use for:** Phase 2+ review/training UIs

---

### CLI Tools (Click)
**Pros:**
- Fast admin tasks
- Scriptable (automation, cron jobs)
- No UI overhead
- Great for SSH/remote access

**Cons:**
- Text-only (no visuals)
- Steeper learning curve for non-technical users

**Use for:** Health monitoring, config management, secrets

---

## API Design (for Web UIs)

All web UIs communicate with backend via REST API:

```python
# marengo/api/routes.py

from flask import Flask, jsonify, request
from marengo.core import database, merge_queue, face_library

app = Flask(__name__)

# Merge Queue
@app.route('/api/merge/queue', methods=['GET'])
def get_merge_queue():
    """List pending merge candidates."""
    pending = merge_queue.get_pending(limit=100)
    return jsonify({'candidates': pending, 'count': len(pending)})

@app.route('/api/merge/candidate/<id>', methods=['GET'])
def get_candidate(id):
    """Get merge candidate details with panels."""
    candidate = merge_queue.get_candidate(id)
    return jsonify(candidate)

@app.route('/api/merge/candidate/<id>/accept', methods=['POST'])
def accept_merge(id):
    """Accept merge, update edges, register face."""
    result = merge_queue.accept(id, user=request.json.get('user'))
    return jsonify(result)

# Face Library
@app.route('/api/faces/library', methods=['GET'])
def get_face_library():
    """List all registered people."""
    people = face_library.list_people()
    return jsonify({'people': people})

@app.route('/api/faces/person/<id>', methods=['GET'])
def get_person(id):
    """Get person details with faces and match history."""
    person = face_library.get_person(id)
    return jsonify(person)

@app.route('/api/faces/person/<id>/rename', methods=['POST'])
def rename_person(id):
    """Rename person (unknown → known name)."""
    new_name = request.json.get('name')
    result = face_library.rename(id, new_name)
    return jsonify(result)

# Edge Visualization
@app.route('/api/edges/camera_pairs', methods=['GET'])
def get_camera_pairs():
    """List all camera pairs with edge counts."""
    pairs = database.get_camera_pairs()
    return jsonify({'pairs': pairs})

@app.route('/api/edges/<src_camera>/<dst_camera>', methods=['GET'])
def get_edges(src_camera, dst_camera):
    """Get 8×8 edge matrix for camera pair."""
    edges = database.get_edges(src_camera, dst_camera)
    return jsonify({'edges': edges})

# Timeline
@app.route('/api/timeline/events', methods=['GET'])
def get_events():
    """List events with filters (date, person, camera, class)."""
    filters = {
        'date': request.args.get('date'),
        'person': request.args.get('person'),
        'camera': request.args.get('camera'),
        'class': request.args.get('class'),
    }
    events = database.get_events(**filters)
    return jsonify({'events': events})

@app.route('/api/timeline/event/<id>/playback', methods=['GET'])
def get_playback(id):
    """Get playback info: manifest, camera transitions, timestamps."""
    playback = database.get_playback_info(id)
    return jsonify(playback)
```

---

## Consolidation Strategy (Phase 4+)

**Option A: Keep Separate Apps**
- Pros: Simple, focused, easy to maintain
- Cons: Multiple browser tabs, no unified view

**Option B: Unified Dashboard (Phase 5)**
```
┌─────────────────────────────────────────────────────┐
│ MarengoCam Dashboard                                │
├────┬────────────────────────────────────────────────┤
│    │ Timeline                                       │
│ T  │ ─────────────────────────────────────────────  │
│ i  │ [Events list...]                               │
│ m  │                                                 │
│ e  ├────────────────────────────────────────────────┤
│ l  │ Merge Queue (3 pending)                        │
│ i  │ ─────────────────────────────────────────────  │
│ n  │ [Candidate preview...]                         │
│ e  │                                                 │
│    ├────────────────────────────────────────────────┤
│ M  │ Face Library (12 people)                       │
│ e  │ ─────────────────────────────────────────────  │
│ r  │ [Face gallery...]                              │
│ g  │                                                 │
│ e  ├────────────────────────────────────────────────┤
│    │ System Health                                   │
│ F  │ ─────────────────────────────────────────────  │
│ a  │ CPU: 42%  Cameras: 20/20  Detections: 12/s    │
│ c  │ [Metrics...]                                   │
│ e  │                                                 │
│    └────────────────────────────────────────────────┘
│ H
│ e
│ a
│ l
│ t
│ h
└────
```

**Decision:** Defer until Phase 4+; separate apps work fine for development.

---

## Security & Access Control

**Phase 0-2 (Development):**
- No authentication required (localhost only)
- UIs bind to `127.0.0.1` (not accessible from network)

**Phase 3+ (Multi-User):**
- Add basic HTTP auth (username/password)
- HTTPS for remote access (self-signed cert acceptable)
- Role-based access: admin vs. reviewer vs. viewer

**Phase 5 (Production):**
- OAuth/SAML integration (optional)
- Audit logs (who accepted/rejected merges)
- Rate limiting (prevent abuse)

---

## Open Items

1. **Mobile-Friendly UI?**
   - Current plan: desktop web UIs work on phone browsers
   - Native mobile app: Phase 5+ (if needed)

2. **Real-Time Updates?**
   - WebSockets for live updates (merge queue, timeline)?
   - Or simple polling (refresh every 5s)?
   - Decision: Polling for Phase 2-3, WebSockets if needed later

3. **Video Player Performance?**
   - Reconstructing MP4 from 10 FPS JPEGs on-demand vs. pre-render?
   - Decision: On-demand for Phase 4 (simpler), pre-render if slow

4. **Desktop App Distribution?**
   - Package tkinter apps as .exe (PyInstaller)?
   - Or assume Python installed?
   - Decision: Python scripts fine for Phase 0-1 (developer only)

5. **UI Framework Upgrade?**
   - Use React/Vue for web UIs in Phase 5?
   - Decision: Plain HTML/JS sufficient for Phase 2-4, reconsider if complexity grows

---

## Summary

**Development Strategy:**
- ✅ Phase 0: Update Camera Grid Viewer to 32×32, build CLI Health Monitor
- 🔨 Phase 1: Build Single-Camera Debug Viewer (tkinter)
- 🔨 Phase 2: Build Merge Decision Queue (web)
- 🔨 Phase 3: Build Edge Dashboard + Face Library Manager (web)
- 🔨 Phase 4: Build Event Timeline Viewer (web)
- ⏳ Phase 5: Consider unified dashboard or Configuration Studio (deferred)

**Separation Benefits:**
- Each UI focused on one task
- Easy to debug and test independently
- Progressive rollout (build UIs as features mature)
- No premature complexity (avoid building unified dashboard before workflow proven)

**Consolidation Later:**
- Phase 4+: Re-evaluate if unified dashboard needed
- Never consolidate just for sake of consolidation
- Keep separation if it works better for users
