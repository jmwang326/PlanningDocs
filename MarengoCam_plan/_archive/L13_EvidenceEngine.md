# Level 13 - Evidence Engine - Technical Specification

## 1. Purpose
This document provides the technical specifications for the **Evidence Engine**. It details the key data structures and function signatures required to implement the tactical plan outlined in `L3_EvidenceEngine.md`.

## 2. Core Data Structures

### `EvidenceRequest`
A simple structure to request an evidence report for a pair of tracks.

```python
@dataclass
class EvidenceRequest:
    source_track_id: int
    candidate_track_id: int
```

### `EvidenceReport`
The structured report returned by the engine. It contains facts, not decisions.

```python
@dataclass
class EvidenceReport:
    request: EvidenceRequest
    
    # AI-driven evidence (from Inference Manager)
    face_similarity: Optional[float] = None
    reid_similarity: Optional[float] = None
    color_similarity: Optional[float] = None
    
    # Static/Config-driven evidence
    portal_match: bool = False
    portal_id: Optional[str] = None
    time_delta_seconds: float = 0.0
    is_within_portal_window: bool = False
    
    # Metadata
    evidence_gathering_duration_ms: float = 0.0
    errors: List[str] = field(default_factory=list)
```

## 3. Key Function Signatures

### `EvidenceEngine` Class

```python
class EvidenceEngine:
    def __init__(self, inference_manager: InferenceManager, portal_config: Dict):
        """
        Initializes the engine with dependencies.
        
        Args:
            inference_manager: A client for the Inference Manager.
            portal_config: The loaded portal configuration for fast lookups.
        """
        self.inference_manager = inference_manager
        self.portal_config = portal_config

    async def gather_evidence(self, request: EvidenceRequest) -> EvidenceReport:
        """
        The main public method. It orchestrates the parallel gathering of all
        evidence for the given track pair.
        """
        report = EvidenceReport(request=request)
        start_time = time.time()

        # Initiate all evidence gathering tasks concurrently
        ai_evidence_task = asyncio.create_task(self._get_ai_evidence(request))
        static_evidence_task = asyncio.create_task(self._get_static_evidence(request))

        # Await results
        ai_results = await ai_evidence_task
        static_results = await static_evidence_task

        # Populate the report
        report.face_similarity = ai_results.get('face_similarity')
        report.reid_similarity = ai_results.get('reid_similarity')
        # ... and so on for all other fields

        report.evidence_gathering_duration_ms = (time.time() - start_time) * 1000
        return report

    async def _get_ai_evidence(self, request: EvidenceRequest) -> Dict:
        """
        Makes parallel calls to the Inference Manager for all AI-based evidence.
        """
        face_task = self.inference_manager.request(
            'face_similarity', {'track1': request.source_track_id, 'track2': request.candidate_track_id}
        )
        reid_task = self.inference_manager.request(
            'reid_similarity', {'track1': request.source_track_id, 'track2': request.candidate_track_id}
        )
        
        results = await asyncio.gather(face_task, reid_task, return_exceptions=True)
        
        return {
            'face_similarity': results[0] if not isinstance(results[0], Exception) else None,
            'reid_similarity': results[1] if not isinstance(results[1], Exception) else None,
        }

    async def _get_static_evidence(self, request: EvidenceRequest) -> Dict:
        """
        Performs fast, local lookups for portal and timing information.
        This requires querying the database for track start/end times and cells.
        """
        # 1. Fetch track details from the database
        # 2. Check for a matching portal in self.portal_config
        # 3. Calculate time delta and check against portal window
        # ...
        return {
            'portal_match': True, # or False
            'is_within_portal_window': True, # or False
            # ...
        }
```
