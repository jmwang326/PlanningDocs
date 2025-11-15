# Level 13 - Panel Assembly Service - Technical Specification

## 1. Purpose
This document provides the technical specifications for the **Panel Assembly Service**. It details the key data structures and function signatures required to implement the tactical plan outlined in `L3_PanelAssemblyService.md`.

## 2. Core Data Structures

### `PanelRequest`
A simple structure to request the creation of a review panel.

```python
from typing import List, Tuple

@dataclass
class PanelRequest:
    merge_id: int
    source_track_id: int
    dest_track_id: int
    panel_type: str = "2x6_comparison" # Allows for future panel types
```

### `PanelResult`
The result returned by the service.

```python
@dataclass
class PanelResult:
    request: PanelRequest
    panel_path: str # Absolute path to the generated image
    success: bool
    error_message: str = None
```

## 3. Key Function Signatures

### `PanelAssemblyService` Class

```python
class PanelAssemblyService:
    def __init__(self, storage_root: str):
        """
        Initializes the service with the root path for media storage.
        
        Args:
            storage_root: The absolute path to the 'storage/' directory.
        """
        self.storage_root = storage_root

    async def create_panel(self, request: PanelRequest) -> PanelResult:
        """
        The main public method to generate a panel. It orchestrates the
        fetching of frames, composition, and saving of the final image.
        """
        # ... implementation ...

    async def _get_keyframes_for_track(self, track_id: int, num_frames: int = 6) -> List[Dict]:
        """
        Queries the database for the most representative keyframes for a track.
        
        Returns:
            A list of detection records (dictionaries) for the keyframes.
        """
        # ... implementation ...

    async def _get_best_face_for_track(self, track_id: int) -> Optional[Dict]:
        """
        Queries the database to find the highest-quality face detection for a track.
        """
        # ... implementation ...

    def _compose_panel(self, track_a_frames: List[np.ndarray], track_b_frames: List[np.ndarray]) -> np.ndarray:
        """
        Takes two lists of cropped images and composes them into a single
        2x6 panel image. Handles resizing and arrangement.
        """
        # ... implementation ...

    def _crop_and_resize(self, frame_path: str, bbox: Tuple[int, int, int, int], target_size: Tuple[int, int], upscale: bool = False) -> np.ndarray:
        """
        Loads a frame from disk, crops the specified bounding box, and resizes it.
        Optionally upscales the image, which is useful for small face crops.
        """
        # ... implementation ...

    def _save_panel(self, merge_id: int, panel_image: np.ndarray) -> str:
        """
        Saves the final composite panel image to the correct directory
        (e.g., 'storage/panels/merge_{id}/panel.jpg') and returns the path.
        """
        # ... implementation ...
```

## 4. Image Processing Dependencies
- **`opencv-python`**: For all core image manipulation tasks (reading, cropping, resizing, writing).
- **`numpy`**: For representing image data in arrays.
- **`pillow`**: Can be used as an alternative or supplement to OpenCV for certain tasks like adding text overlays.

## 5. Database Interaction
The service's only interaction with the database is read-only. It needs to query the `detections` table to find the paths and bounding boxes of keyframes and high-quality faces associated with the requested `track_id`s. It does not write any data to the database.