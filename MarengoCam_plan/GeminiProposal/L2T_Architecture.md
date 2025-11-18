# L2T: Physical Architecture & Tech Stack

This document defines the concrete technology stack and deployment architecture for the MarengoCam system.

---

## 1. Technology Stack

The system is built on a modern, containerized web service architecture.

*   **Language:** Python 3.11+ (Backend), TypeScript/React (Frontend).
*   **Hardware:** NVIDIA Blackwell (or equivalent) for GPU acceleration.
*   **Containerization:** Docker & Docker Compose.
*   **Source Control:** Git.

---

## 2. Service Architecture

The system is composed of three primary services and two external dependencies, defined in a single `docker-compose.yml`.

### 2.1 Core Services

1.  **`marengo-engine` (Backend)**
    *   **Role:** The monolithic application core. Runs the Conductor, API, and all Logic components.
    *   **Framework:** **FastAPI** (Python). Chosen for high-performance async I/O and auto-generated API docs.
    *   **Responsibilities:**
        *   `FrameFetcher`: Async polling of camera streams.
        *   `Conductor`: The main orchestration loop.
        *   `API`: REST endpoints for the GUI.
    *   **Access:** Has direct access to the GPU. Mounts the `/data/frames` volume.

2.  **`marengo-gui` (Frontend)**
    *   **Role:** The user interface.
    *   **Framework:** **React** (served via Nginx).
    *   **Responsibilities:** Renders the Timeline, Review Queue, and Dashboards. Consumes the Backend API.

3.  **`postgres-db` (Database)**
    *   **Role:** Persistent metadata storage.
    *   **Technology:** **PostgreSQL**.
    *   **Schema:** Stores `GlobalAgents`, `LocalTracks`, `GridLearnings`, etc.

### 2.2 External Dependencies

1.  **`codeproject-ai`**
    *   **Role:** Local inference server for Face Recognition and ALPR.
    *   **Integration:** Accessed via REST API by the `marengo-engine`.

2.  **External LLM Provider**
    *   **Role:** Visual adjudication (Cloud-based).
    *   **Integration:** Accessed via HTTPS API.

---

## 3. Data Layout

The system uses a hybrid storage model to balance performance and cost.

*   **Structured Data (SQL):**
    *   All logic-related metadata (Track start/end times, Agent IDs, Config) lives in PostgreSQL.
*   **Unstructured Data (File System):**
    *   **Raw Frames:** Stored as JPEGs in `/data/frames/{camera_id}/{timestamp}.jpg`.
    *   **Video Chunks:** Stored as MP4s in `/data/chunks/{camera_id}/{timestamp}.mp4`.
    *   **Why:** Storing binary media in a database bloats backups and slows queries. The file system is the most efficient store for this data.

