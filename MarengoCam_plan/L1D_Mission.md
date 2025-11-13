# Level 1D - Mission Discourse

## Purpose
Brief rationale for key mission decisions and tradeoffs.

## Why Agent-Centric vs Detection-Centric?

**Decision:** Track persistent agents (people/vehicles), not just detections.

**Rationale:**
- User cares about "what did Person A do today?" not "what did CamA see?"
- Timeline reconstruction requires linking observations across cameras
- Persistent identity survives occlusions, camera transitions

**Tradeoff:**
- More complex (need merge logic, identity management)
- But: more useful output (agent timeline vs disconnected clips)

---

## Why Observable Evidence Only?

**Decision:** No statistical priors like "Person A always drives Car B."

**Rationale:**
- Routines change (vacation, new car, visitor drives your car)
- Statistical assumptions fail on edge cases (exactly when you need the system most)
- Observable evidence is debuggable ("face match" vs "P(A|B) = 0.85")

**Tradeoff:**
- Can't leverage patterns for disambiguation
- But: more robust to unusual activity, easier to debug

---

## Why Face Recognition Before Re-ID?

**Decision:** Prioritize CodeProject.AI face recognition in Phase 2, defer Re-ID to Phase 4-5.

**Rationale:**
- Face recognition is mature, trainable, already available
- Handles 60%+ of merge cases (known people)
- Re-ID useful for vehicles and face-occluded people, but not required for initial rollout

**Tradeoff:**
- Vehicles require LLM adjudication until Re-ID available
- But: faster path to automation (face library builds quickly)

---

## Why LLM as Fallback, Not Primary?

**Decision:** Use Gemini Flash 2.0 only when face/time/space insufficient.

**Rationale:**
- Time/space gating is deterministic, fast, free
- Face recognition is faster than LLM panel assembly
- LLM best for ambiguous cases (clothing similarity, partial occlusion)

**Tradeoff:**
- LLM could handle more cases if used earlier
- But: cost, latency, and debuggability favor deterministic methods first

---

## Why 3.0 Second Threshold for "Real" Agent?

**Decision:** Agent must move for ≥ 3.0 seconds to promote camera to Active state.

**Rationale:**
- Filters noise (wind, shadows, brief detector glitches)
- High confidence that object is real and moving
- 3.0s = 6-30 frames at 2-10 FPS (plenty for confirmation)

**Tradeoff:**
- Misses very brief legitimate activity (person quickly grabs package)
- But: rare, and pre-roll buffer may still capture it

**Open question:** Should threshold vary by class? (person 3.0s, vehicle 5.0s, animal 2.0s?)

---

## Why 10 FPS Target (Not Higher)?

**Decision:** Acquire frames at 10 FPS, not 15 or 30.

**Rationale:**
- 10 FPS smooth enough for playback (100ms per frame)
- Face recognition and portal crossing benefit from higher temporal resolution
- GPU budget: 10 cameras × 10 FPS = 100 FPS sustained inference at Active
- Storage: 10 FPS × 9 days × 10 cameras = manageable

**Tradeoff:**
- Higher FPS = smoother playback, better tracking
- But: GPU/storage cost, diminishing returns above 10 FPS for this use case

**Note:** Starting at 2 FPS (Phase 0-1) to prove system on limited resources. Increase to 10 FPS in Phase 2 when face recognition and portals benefit from higher frame rate.

---

## Why Local-Only (No Cloud)?

**Decision:** Everything runs locally except Gemini Flash 2.0 API calls.

**Rationale:**
- Privacy (surveillance footage stays on-premises)
- No internet dependency for core functionality
- Cost (no cloud storage fees)

**Tradeoff:**
- No remote access (must VPN to view)
- But: acceptable for residential use

---

## Why SQLite (Not PostgreSQL)?

**Decision:** Use SQLite for metadata storage.

**Rationale:**
- Single-user system (no concurrent writes from multiple servers)
- Embedded, no separate database server
- Excellent read performance for timeline queries
- Proven at scale (metadata << frame storage)

**Tradeoff:**
- Limited write concurrency
- But: acceptable for single-process ingest + occasional queries

---

## Change Log

**2025-11-13**: Initial mission definition extracted from BIGPICTURE_V2.md during framework reorganization.
