# L3: System Health

## Component: `SystemHealth`

## Purpose

The `SystemHealth` component ensures the system remains stable and responsive, even under heavy load or failure conditions. It is responsible for monitoring critical metrics and triggering "Survival Mode" when necessary.

## Functional Requirements

### 1. Performance Monitoring
**Goal:** Detect when the system is falling behind reality.

*   **Requirement:** The system must track **Processing Lag** (Time difference between `Event Occurred` and `Event Processed`).
*   **Requirement:** The system must track **Queue Depth** (Number of pending items in the Review Queue).

### 2. Automated Remediation (Survival Mode)
**Goal:** Prevent a total system crash during overload (e.g., "The Party" scenario).

*   **Requirement:** If `Processing Lag` exceeds a critical threshold (e.g., 30s), the system must automatically degrade fidelity to recover speed.
    *   *Action:* Reduce YOLO frame rate (e.g., 10fps -> 2fps).
    *   *Action:* Disable expensive Re-ID embedding generation for low-confidence tracks.
*   **Requirement:** If `Queue Depth` exceeds a critical threshold (e.g., 100 items), the system must stop adding low-value items.
    *   *Action:* Auto-archive "Unknown/Uncertain" tracks without human review.

### 3. Failure Detection
**Goal:** Detect hardware or network failures.

*   **Requirement:** The system must detect if a camera stream has stopped providing frames ("Sensor Blackout").
*   **Requirement:** The system must detect if the GPU is unreachable or stuck.
