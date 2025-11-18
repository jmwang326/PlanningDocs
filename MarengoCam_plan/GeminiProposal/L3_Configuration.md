# L3: System Configuration

## Component: `Configuration`

## Purpose

The `Configuration` component provides a centralized, human-readable, and dynamically updatable source of truth for all system parameters. It governs the behavior of every other component, from camera stream URLs to the confidence thresholds used in identity resolution.

The entire system is designed to be configured via a single `marengo.yaml` file, allowing for easy management, versioning, and deployment.

## High-Level Concepts

### Operational Stages

The system is designed to operate in distinct stages, each with its own configuration profile to optimize for the current goal (e.g., learning vs. autonomous operation).

1.  **`PROD_LEARNING`**: A conservative stage focused on bootstrapping the system's intelligence. Merging is cautious, and human review is prioritized to ensure the `6x6 Grid` and face libraries are trained on clean, high-quality data.
2.  **`PROD_AUTO`**: An autonomous stage where the system operates with a higher degree of confidence. Merging is more aggressive, and the system relies more on its learned models and LLM adjudication, minimizing the need for human intervention.

### Dynamic Hot-Reloading

The system is designed to be highly available and responsive to configuration changes without requiring a full restart. The `marengo.yaml` file is monitored for changes, which are then validated and applied gracefully to the running system. This allows for on-the-fly tuning of thresholds and parameters.

### Secrets Management

To maintain security, sensitive information (e.g., API keys, passwords, URLs) is not stored directly in the main configuration file. It is managed separately in an encrypted `secrets.yaml` file and referenced in the main configuration using a `!secret` tag.

## Output

The `Configuration` component provides a validated `SystemConfig` object that is accessible to all other components in the system.
