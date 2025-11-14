# Level 3 - Panel Assembly Service - Tactical

## 1. Purpose

The **Panel Assembly Service** is a specialized image processing component with a single, clear responsibility: to create standardized visual artifacts (panels) for review by either an LLM or a human. It abstracts the complexities of image retrieval and manipulation from the logical decision-making components of the system.

## 2. Key Responsibilities

- **Visual Artifact Generation:** The service's primary function is to generate composite images, specifically the `2x6` panels used for merge adjudication. This involves:
    - Querying the `detections` table to find the best keyframes for a given track.
    - Loading the raw JPEG frame data from the file storage.
    - Cropping the agent's bounding box from each frame.
    - Resizing and arranging the crops into a standardized grid format (e.g., a `2x6` panel).
    - Extracting and cropping the highest-quality face detections for inclusion in the panel.
- **Image Manipulation:** Handle necessary image processing tasks like resizing, upscaling (especially for face crops to ensure clarity), and compositing multiple images into a single JPEG.
- **Storage:** Save the final generated panel to the appropriate location in the `storage/panels/` directory, following the system's naming conventions.
- **Stateless Operation:** The service is stateless. It receives a request with track IDs and returns the path to the generated image file. It does not hold any state between requests.

## 3. Operational Flow

1.  **Request:** The `Evidence Engine`, after deciding a merge requires review, calls the `Panel Assembly Service` with the source and destination track IDs.
2.  **Frame Selection:** The service queries the database for the most representative `detections` (keyframes) associated with each track. It will select the first, last, and a few evenly-spaced keyframes in between.
3.  **Image Retrieval & Processing:** For each selected keyframe, the service:
    - Reads the JPEG from the path specified in the `detections` record.
    - Crops the bounding box.
    - If a high-quality face is available, it may perform an additional, tighter crop on the face and upscale it for clarity.
4.  **Composition:** The service creates a new blank image and composites the processed crops into the final `2x6` panel layout.
5.  **Storage & Response:** The final JPEG panel is saved to a unique path (e.g., `storage/panels/merge_12345/panel.jpg`), and this file path is returned to the `Evidence Engine`.

## 4. Strategic Importance

By isolating all image manipulation into this single service, the rest of the system remains clean. The `Evidence Engine` can focus entirely on logic and rules, and the `Inference Manager` can focus on routing. This separation makes the components easier to develop, test, and maintain. It also allows the `Panel Assembly Service` to be optimized independently, for instance by using a more efficient image processing library, without affecting any other part of the system.