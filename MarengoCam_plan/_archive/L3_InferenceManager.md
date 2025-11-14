# Level 3 - Inference Manager - Tactical

## 1. Purpose

The **Inference Manager** is the central nervous system for all AI-related tasks. Its primary purpose is to abstract the complexity of using multiple, heterogeneous AI services (local GPUs, remote APIs, specialized models, and even human reviewers) and present them as a single, unified capability to the rest of the system. It is responsible for routing inference requests to the most appropriate provider, managing failures, and ensuring that the system can flexibly leverage the best available resource for any given task.

## 2. Key Responsibilities

- **Service Registry:** Maintain a dynamic registry of all available inference providers, loaded from the central configuration. This includes their capabilities (e.g., `person_detection`, `face_recognition`, `adjudication`), endpoints, and priorities.

- **Intelligent Routing:** Route incoming inference requests to the best-suited provider. The routing logic will consider:
    - **Request Type:** A `face_recognition` request goes to a CodeProject.AI instance, while a `merge_adjudication` request might go to a Gemini LLM or the human review queue.
    - **Priority:** Preferentially use high-performance, low-cost local resources (like the RTX 5070 Ti for primary detection) before falling back to remote or lower-priority services.
    - **Health Status:** Only route requests to services that are currently reported as healthy.

- **Health Checking & Failover:** Proactively monitor the health of all registered services.
    - Implement a circuit breaker pattern: after a configurable number of consecutive failures, a service is temporarily marked as "unhealthy" and taken out of the routing pool.
    - Periodically attempt to reconnect to unhealthy services to see if they have recovered.
    - This ensures that a single failing service does not bring down the entire system or introduce significant latency.

- **Unified Interface:** Provide a simple, consistent API to the rest of the application (e.g., to the `Track State Manager` or `Evidence Engine`). Callers will use a simple function like `InferenceManager.request(task_type, data)` without needing to know which specific AI model or API will handle it.

## 3. Operational Flow

1.  **Initialization:** On startup, the Inference Manager loads the `services.yaml` configuration, registers all defined providers, and initiates background health checks for each one.

2.  **Request:** Another component, such as the `Track State Manager`, needs to perform an inference task (e.g., run object detection on a new frame). It calls `InferenceManager.request('detection', frame_data)`.

3.  **Routing:** The Inference Manager consults its registry. It identifies all healthy providers that have the `detection` capability, sorts them by priority, and selects the top one.

4.  **Execution & Failover:** It dispatches the request to the selected provider (e.g., a local YOLO11 instance).
    - If the request is successful, the result is returned to the caller.
    - If the request fails (timeout, error code), the manager logs the failure, updates the health status for that provider (incrementing the failure count for the circuit breaker), and immediately attempts the request again with the next-highest-priority provider.

5.  **Adjudication Special Case:** When a merge decision requires higher-level reasoning, a request is sent for `adjudication`. The Inference Manager can route this to the Gemini LLM API. If the LLM's confidence is low, or for periodic auditing, it can route the *same request* to the "Human-in-the-Loop" provider, which simply adds the item to the GUI's review queue. This elegantly treats human reviewers as another asynchronous, managed inference endpoint.

## 4. Strategic Importance

By centralizing inference logic, the system becomes incredibly modular and future-proof. Adding a new, more powerful object detection model, a different cloud-based face recognition service, or even a specialized vehicle analysis model becomes a simple matter of adding an entry to the `services.yaml` file, with no changes required to the core application logic. This architecture allows the system to adapt to new technology and changing hardware resources with minimal effort.
