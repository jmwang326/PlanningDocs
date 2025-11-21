# 08 - Component: Dashboard (The Face)

## Purpose
The user interface for investigation, review, and configuration.

**Philosophy:**
- **Investigation First:** The primary use case is "What happened?"
- **Review Second:** The secondary use case is "Teach the system."
- **Config Third:** Settings are for Admins only.

---

## 1. Page Specifications

### 1.1. The Trace Explorer (Home)
**Goal:** Visualize movement across time and space.
- **Widgets:**
    - **Timeline (Gantt):** Rows = Cameras. Bars = Tracks. Color = Entity ID.
    - **Map (Grid):** 6x6 Grid overlay showing active cells.
    - **Filter Bar:** Time Range, Entity ID, Object Type.
- **Interaction:** Click a Bar -> Open "Entity Detail" modal.

### 1.2. The Inbox (Review Queue)
**Goal:** Rapidly process ambiguous merges.
- **Layout:** Split View.
    - **Left:** Candidate Track (Video Loop).
    - **Right:** Potential Match (Video Loop/Face Crop).
    - **Center:** Confidence Score & Reason ("Spatial Overlap: 0.4s").
- **Actions:** [Merge (Key: Y)] [Reject (Key: N)] [Skip (Key: Space)].

### 1.3. Entity Manager
**Goal:** Manage the "Rolodex" of known people.
- **Layout:** Grid of "Person Cards".
- **Card:** Best Face Crop + Name + "Last Seen".
- **Action:** Click to rename, merge duplicates, or delete.

### 1.4. System Status
**Goal:** Health at a glance.
- **Widgets:**
    - **Sparklines:** FPS, GPU Mem, Disk Usage.
    - **Log Stream:** Live tail of `audit_logs`.
    - **Grid Heatmap:** Visualizing the learned travel times.

---

## 2. Technology Stack
- **Phase 1:** Streamlit. (Fast iteration, pure Python).
- **Phase 2:** React/Vue + Recharts. (Better performance for complex timelines).
