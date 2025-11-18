# L3: System Health

## Component: `SystemHealth`

## Purpose

The `SystemHealth` component is a passive monitoring and alerting system that runs alongside the main processing pipeline. Its purpose is to provide real-time visibility into the performance and reliability of the entire MarengoCam application, detect issues before they become critical, and trigger alerts or automated remediation actions.

## High-Level Process

The `SystemHealth` monitor continuously collects metrics and logs from all other components. It aggregates this data to track key performance indicators (KPIs) and compares them against predefined thresholds.

## Key Monitored Metrics

The health of the system is understood by monitoring a few critical KPIs:

1.  **Processing Lag:** The delay between an event happening on camera and it being fully processed. This is the primary indicator of overall system load and throughput.

2.  **Queue Depths:** The number of items waiting in the `IdentityResolver`'s merge queue and the `EvidenceProcessor`'s review queue. This indicates the scale of uncertainty the system is dealing with.

3.  **Auto-Merge Rate:** The percentage of tracks that are merged automatically without needing human or LLM review. This is the primary measure of the system's intelligence and learning progress.

4.  **Resource Utilization:** Metrics such as GPU utilization, disk I/O, and database query performance. These help identify hardware or infrastructure bottlenecks.

## Core Features

-   **Health Dashboard:** A web-based UI that provides a real-time and historical view of all key metrics, allowing an operator to quickly assess the system's status.

-   **Alerting:** A multi-level alerting system (Info, Warning, Critical) that sends notifications through various channels (logs, email) when metrics deviate from their target ranges.

-   **Automatic Remediation:** A set of self-healing rules that can take action to prevent cascading failures during load spikes, such as by dynamically reducing the processing frame rate.
