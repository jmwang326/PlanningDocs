# Level 13 - Inference Manager

## 1. Purpose
This document provides the technical specifications for the **Inference Manager**. This component is the central router for all AI-related tasks, directing requests to the appropriate specialized service (e.g., local YOLO, CodeProject.AI) and managing their health.

---

## 2. Core Responsibility: Hybrid AI Strategy
The Inference Manager implements a hybrid AI strategy to leverage the best tool for each specific task. It does not rely on a single, monolithic AI model.

-   **General Object Detection (YOLO):** All primary object detection and tracking tasks for agents like people and vehicles are routed to the local YOLO service. This provides fast, efficient, and local-first processing.

-   **Face Recognition (CodeProject.AI):** All face-related tasks, including recognition and registration, are routed exclusively to the CodeProject.AI (CPAI) service. This is a deliberate architectural choice, as CPAI provides a highly specialized and more accurate model for face recognition than the general-purpose YOLO model. This separation of concerns ensures maximum accuracy for critical identification tasks.

---

## 3. Service Registry and Health Checks
This section will define the data contracts for the service registry, the precise algorithms for the health-check and circuit-breaker logic, and the API for routing requests.
