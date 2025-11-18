# L3T: External AI Contracts

This document provides the technical API contracts for interacting with external AI services.

---

## 1. Face Recognition Service

-   **Provider:** CodeProject.AI (self-hosted)
-   **Purpose:** To recognize known faces and to register new ones.

### Recognize Face

-   **Endpoint:** `POST /v1/vision/face/recognize`
-   **Usage:** Called by the `IdentityResolver` to get a primary piece of evidence for merging.
-   **Request:**
    ```json
    {
      "image": "<base64_encoded_jpeg>",
      "min_confidence": 0.75
    }
    ```
-   **Success Response (Match):**
    ```json
    {
      "success": true,
      "predictions": [
        { "userid": "agent_123", "confidence": 0.87 }
      ]
    }
    ```
-   **Error Handling:**
    -   **Timeout:** 5 seconds.
    -   **Retries:** 3 attempts with exponential backoff.
    -   **On Failure:** The evidence is considered inconclusive. The track is marked as uncertain and may be queued for review.

### Register Face

-   **Endpoint:** `POST /v1/vision/face/register`
-   **Usage:** Called by the `EvidenceProcessor` after a human has validated a merge, adding a new, high-quality face crop to a known agent's library.
-   **Request:**
    ```json
    {
      "image": "<base64_encoded_jpeg>",
      "userid": "agent_123"
    }
    ```

---

## 2. LLM Adjudication Service

-   **Provider:** Anthropic (via API)
-   **Model:** Claude series (e.g., `claude-3-sonnet-20240229`)
-   **Purpose:** To act as an automated reviewer for ambiguous merge cases that are not clear enough for geometric logic but may be obvious to a vision model.

### Adjudicate Merge Decision

-   **Endpoint:** `POST https://api.anthropic.com/v1/messages`
-   **Usage:** Called by the `EvidenceProcessor` for items in the review queue.
-   **Request Summary:**
    -   **Role:** `user`
    -   **Content:** A multi-modal prompt containing:
        1.  An image of the face/body from `Track_A`.
        2.  An image of the face/body from `Track_B`.
        3.  A text block with context (camera names, timing) and a clear instruction to return a JSON object with a `decision`, `confidence`, and `reasoning`.
-   **Success Response:** The `content` block of the API response will contain a text field with the requested JSON object.
    ```json
    {
      "decision": "merge",
      "confidence": "high",
      "reasoning": "Same clothing (blue shirt), similar body build, and plausible timing."
    }
    ```
-   **Error Handling:**
    -   **Timeout:** 30 seconds.
    -   **Retries:** 2 attempts.
    -   **On Failure:** The item remains in the review queue for a human to adjudicate.
