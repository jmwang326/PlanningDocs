# L3T: System Health Technical Details

This document provides the technical thresholds and logic for system stability.

---

## Monitored Metrics & Thresholds

-   **Processing Lag:**
    -   **Target:** < 15s
    -   **Critical:** > 30s (Triggers Survival Mode)

-   **Review Queue Depth:**
    -   **Target:** < 50 tracks
    -   **Critical:** > 100 tracks (Triggers Auto-Archive)

-   **Sensor Status:**
    -   **Critical:** No frames received for > 5 seconds (Triggers Alert)

---

## Survival Mode Logic (Degraded Operation)

The system includes self-healing mechanisms to respond to sustained critical conditions.

### 1. Load Shedding (Trigger: Lag > 30s)
-   **Action:** Reduce `ChunkProcessor` sampling rate.
    -   `Active` Cameras: 10fps -> 5fps
    -   `Armed` Cameras: 2fps -> 0fps (Motion gate only)
-   **Action:** Disable Re-ID embedding generation for tracks with `confidence < 0.5`.

### 2. Queue Management (Trigger: Queue > 100)
-   **Action:** Enable `Auto-Archive`.
    -   Any new track with `class != person` is immediately archived.
    -   Any new track with `duration < 5s` is immediately archived.
    -   Any new track with `face_confidence < 0.4` is marked "Unknown" and archived without review.

### 3. Recovery
-   **Trigger:** Metrics return to `Target` range for > 60 seconds.
-   **Action:** Restore full frame rates and disable Auto-Archive.
