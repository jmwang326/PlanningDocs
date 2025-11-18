# L3T: System Health Technical Details

This document provides the specific technical parameters and logic for the `SystemHealth` component.

---

## Monitored Metrics & Thresholds

-   **Processing Lag:**
    -   **Target:** < 15s
    -   **Warning:** > 15s
    -   **Critical:** > 30s

-   **GPU Utilization:**
    -   **Target:** 70-90%
    -   **Warning:** > 95% (Saturation)

-   **Merge Queue Depth** (`IdentityResolver`):
    -   **Target:** < 50 tracks
    -   **Warning:** > 50 tracks
    -   **Critical:** > 100 tracks

-   **Review Queue Depth** (`EvidenceProcessor`):
    -   **Target:** < 20 tracks
    -   **Warning:** > 20 tracks
    -   **Critical:** > 50 tracks

-   **Auto-Merge Rate:**
    -   **Target:** > 80% (post-bootstrap)
    -   **Warning:** < 60%

-   **Database Query Performance:**
    -   **Target:** < 100ms (average)
    -   **Warning:** > 100ms
    -   **Critical:** > 500ms

---

## Alerting Configuration

-   **Levels:** `Info`, `Warning`, `Critical`.
-   **Channels:**
    -   `log`: All levels.
    -   `dashboard`: `Warning` and `Critical`.
    -   `email`: `Critical` only.
-   **Throttling:** A specific alert (e.g., "High Processing Lag") will be suppressed for 5 minutes after being sent to avoid notification spam.

---

## Automatic Remediation Logic

The system includes self-healing mechanisms to respond to sustained critical conditions.

-   **If `gpu_util > 95%` for > 60 seconds:**
    -   **Action:** Temporarily reduce the inference sampling rate for cameras in `Armed` and `Post` states to shed load and prioritize `Active` cameras.

-   **If `processing_lag > 30s` for > 60 seconds:**
    -   **Action:** Aggressively prioritize `Active` cameras by temporarily skipping all processing for `Armed` and `Post` state cameras.

-   **If `merge_queue_depth > 100` for > 5 minutes:**
    -   **Action:** Increase the batch processing frequency of the `IdentityResolver` to work through the backlog more quickly.

---

## Startup Health Checks

Before the main application starts, a series of blocking health checks are performed to ensure essential dependencies are available.

-   **Database Connection:**
    -   **Action:** Attempt to connect to the configured database.
    -   **Retry:** If the connection fails, retry every 5 seconds for a total of 60 seconds.
    -   **Failure:** If no connection can be established after 60 seconds, log a fatal error and exit with a non-zero status code.

-   **Configuration Validation:**
    -   **Action:** Load and validate `marengo.yaml` and `secrets.yaml`.
    -   **Failure:** If validation fails, log a fatal error detailing the misconfiguration and exit.

-   **External AI Services:**
    -   **Action:** Ping the configured endpoints for critical services (e.g., CodeProject.AI).
    -   **Failure:** If a critical service is unavailable, log a `Warning`. The system will still start but will operate in a degraded mode.
