# 06 - Implementation Plan (The Schedule)

## Purpose
Defines the step-by-step execution plan to build MarengoCam. It follows the **"Vertical Slice"** strategy, ensuring a runnable system at every stage.

---

## 1. Evolutionary Contract Reference

| Component | Stage 1 (Skeleton) | Stage 2 (The Eye) | Stage 3 (Handshake) | Stage 4 (Network) | Stage 5 (Brain) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ChunkProcessor** | Blind (No GPU). Reads video, writes metadata. | Sighted (YOLO). Writes `LocalTracks`. | Sighted. Adds Evidence extraction. | Full. Adds Face Rec (AI). | Full. Optimized (Motion Gate). |
| **IdentityResolver** | Stub. Always "New ID". | Stub. | Temporal Linker (Same Cam). | Spatial Linker (Hardcoded Grid). | Dynamic Linker (Learned Grid). |
| **Data Model** | Schema Defined. DB Up. | `local_tracks` populated. | `global_entities` populated. | `grid_links` read-only. | `grid_links` read-write. |
| **Dashboard** | None. | Raw Table View. | Timeline View. | Review Queue (Manual Merge). | Full Analytics. |
| **Config** | Static YAML. | Static YAML. | Static YAML. | DB-Backed (Read). | DB-Backed (Hot Reload). |

---

## 2. Phased Rollout

### Phase 1: The Skeleton (Weeks 1-2)
**Goal:** End-to-end plumbing. Video file -> Script -> Database.
- **Tasks:**
    1.  Spin up Postgres & Redis in Docker.
    2.  Implement `01_DataModel` (SQL Schema).
    3.  Build `ChunkProcessor` (Stage 1 - Blind).
    4.  Verify: Can query DB and see "processed chunks".

### Phase 2: The Eye (Weeks 3-4)
**Goal:** See what is happening.
- **Tasks:**
    1.  Enable GPU/YOLO in `ChunkProcessor` (Stage 2).
    2.  Implement `08_Dashboard` (Trace View - Read Only).
    3.  Verify: Can search for "Person" at "10:00 AM".

### Phase 3: The Handshake (Weeks 5-6)
**Goal:** Track individuals on a single camera.
- **Tasks:**
    1.  Implement `IdentityResolver` (Stage 3 - Temporal).
    2.  Implement `07_API` (Merge Endpoint).
    3.  Implement `08_Dashboard` (Review Queue).
    4.  Verify: Person standing still is 1 Track ID.

### Phase 4: The Network (Weeks 7-8)
**Goal:** Track across cameras (Manual/Hardcoded).
- **Tasks:**
    1.  Implement `IdentityResolver` (Stage 4 - Spatial Hardcoded).
    2.  Enable CodeProject.AI (Face Rec).
    3.  Verify: Person walking A->B is 1 Global ID.

### Phase 5: The Brain (Weeks 9+)
**Goal:** System learns and automates.
- **Tasks:**
    1.  Enable `IdentityResolver` (Stage 5 - Dynamic Learning).
    2.  Enable `04_SystemHealth` (Survival Mode).
    3.  Enable `05_Config` (Hot Reload).
    4.  Verify: Grid stats update after manual merges.
