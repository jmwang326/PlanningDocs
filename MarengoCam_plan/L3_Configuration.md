# Level 3 - Configuration

## Purpose
This document defines the tactical plan for the **Configuration System**. It describes how configuration data (camera settings, portals, thresholds) is managed and provided to the system. This includes the processes for hot-reloading and secrets management.

## Operational Mode Configurations
This section details the key configuration differences for the two live operational modes. A single `mode` parameter in the master configuration file will control these settings.

### 1. Online Learning Mode
**Purpose:** To run the live system conservatively, gathering ground-truth data and building trust while minimizing incorrect automatic decisions.

**Key Parameters:**
*   `mode: "online_learning"`
*   `merging.auto_merge_threshold: 0.85` (High threshold, requires very strong evidence for an automatic merge)
*   `merging.human_review_queue: true` (Medium-confidence merges are flagged and sent to a queue for human validation)
*   `external_ai.face_recognition.auto_register_threshold: 0.95` (Extremely high bar for automatically adding a new face to the library without human sign-off)

### 2. Autonomous Mode
**Purpose:** To allow the mature, trained system to operate with a higher degree of autonomy, reducing the need for human oversight.

**Key Parameters:**
*   `mode: "autonomous"`
*   `merging.auto_merge_threshold: 0.70` (Lowered threshold, system is more confident in its merge decisions)
*   `merging.human_review_queue: false` (Only errors or very low-confidence events are flagged for review)
*   `external_ai.face_recognition.auto_register_threshold: 0.75` (Lowered bar for automatically registering new faces that are seen consistently)

## Tactical Plan: Hot-Reloading

The system must be able to apply configuration changes without a full restart. This process is managed by a dedicated configuration watcher.

1.  **Watch for Changes:** A background process monitors the modification time of the master `marengo.yaml` file every 5 seconds.
2.  **Validate New Config:** Upon detecting a change, the system loads the new YAML file into a temporary data structure. It is validated against the schema defined in `L13_Configuration.md`. If validation fails, the change is rejected, an error is logged, and the old configuration remains active.
3.  **Compute Diff:** The system compares the new, validated config with the currently active config to identify what specific values have changed.
4.  **Apply Changes Gracefully:** The configuration manager publishes the changes to all subscribed components.
    *   **Thresholds & simple values:** Applied immediately.
    *   **Camera settings:** The `Video Ingestor` will adjust frame rates or resolutions on its next cycle.
    *   **External AI Services:** The `External AI Services` manager will gracefully drain requests from any removed services and begin routing to new or modified ones.
5.  **Safety Rollback:** If, within 60 seconds of applying a new configuration, the system's overall error rate spikes (e.g., by >20%), the system will automatically revert to the last known good configuration and log a critical alert.

## Tactical Plan: Secrets Management

To avoid storing sensitive information in the main configuration file, secrets are managed separately.

1.  **External Secrets File:** A file named `secrets.yaml` (or as defined in `secrets_file`) will store sensitive data like API keys and passwords. This file should be encrypted on disk (e.g., using DPAPI on Windows or AES cross-platform) and excluded from version control.
2.  **Placeholder Syntax:** The main `marengo.yaml` file will reference secrets using the `!secret` tag (e.g., `password: !secret bi_password`).
3.  **Runtime Resolution:** When the configuration is loaded into memory, the configuration loader is responsible for reading the secrets file, decrypting it, and injecting the secret values into the configuration data structure where the placeholders are found. The plaintext secrets should only ever exist in memory.
4.  **CLI for Management:** A command-line interface (`marengo secrets set <key>`) will be provided to securely add or update secrets in the encrypted file without requiring manual decryption/re-encryption.
