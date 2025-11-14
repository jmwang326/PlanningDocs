# L2: Strategy Principles

**Purpose:** To document the foundational strategic principles that guide the design and implementation of the entire system. These principles are the "why" behind the technical choices in the L10+ documents. They ensure that development remains focused on a coherent, debuggable, and evidence-based approach.

---

## 1. Agent-Centric Thinking
**Principle:** Agents are persistent entities (people, vehicles), not transient detections.

- An agent is assumed to exist even when not visible (e.g., inside a building, occluded by an object, or between cameras).
- The system's primary goal is to construct a timeline that follows the agent's journey across the property, not simply to list what each camera saw.
- This means the fundamental question is always "What did Agent A do?" rather than "What did Camera X see?"

---

## 2. Observable Evidence Only
**Principle:** The system must make decisions based only on what it can see, not on what it assumes or predicts.

- **No Statistical Priors:** The system will not use historical patterns like "Person A usually drives Car B" or "The mail carrier arrives at 3 PM" to influence merge decisions. While these patterns are often true, they are brittle and fail in edge cases (a visitor drives the car, a holiday changes the mail schedule).
- **Focus on Verifiable Evidence:** Decisions will be based on concrete, observable evidence such as face recognition matches, vehicle license plates, Re-ID similarity, and the spatio-temporal plausibility of a track moving through a known inter-camera path.
- **Debuggability:** An evidence-based system is debuggable. A failed merge can be traced back to a specific piece of evidence (e.g., "the face match failed because of poor lighting"), whereas a failure in a statistical model is often opaque.

---

## 3. Count Evidence, Don't Do Arithmetic
**Principle:** Merge decisions will be made by counting discrete pieces of strong and weak evidence, not by summing weighted scores.

- **Codifiable and Observable Rules:** The logic is expressed in clear, auditable rules.
  - **GOOD:** `IF (face_match_confidence > 0.8) AND (is_only_candidate) THEN auto_merge`.
  - **BAD:** `score = (0.4 * reid_score) + (0.3 * time_factor) + (0.3 * spatial_factor)`.
- **Why:** Weighted formulas are difficult to tune and impossible to debug. When a merge based on a composite score fails, it's unclear which component was at fault. In contrast, an evidence-counting system allows for precise analysis: "The merge was rejected because it only had 'weak timing evidence' and lacked any 'strong evidence' like a face match."
- **Thresholds, Not Formulas:** The system will use simple, clear thresholds based on the *number* of strong or weak evidence types available for a given merge candidate.

---

## 4. Configuration Over Blind Learning
**Principle:** Leverage the user's ground-truth knowledge of the environment instead of attempting to discover it from scratch.

- The user knows where doors, gates, and paths are. This information should be manually configured as `Portals` in the system.
- The system's learning task is to determine the *properties* of these configured paths (e.g., "how long does it take to walk from the driveway to the front door?"), not to statistically infer that a path exists in the first place.
- This approach is more efficient and far less error-prone than trying to have the system discover the fundamental layout of the property through unsupervised learning.

---

## 5. Human-in-the-Loop as the Foundation for Automation
**Principle:** Trust in automation must be earned through a gradual, human-supervised process.

- The system will initially rely on a human operator to validate all ambiguous merge decisions.
- These human-validated decisions serve two purposes:
  1. They provide the immediate ground truth for correcting the agent timeline.
  2. They create a high-quality training dataset for the system's learning components (Face Recognition, Re-ID, and `6x6` grid timings).
- Automation will be phased in gradually, starting with the most high-confidence scenarios (e.g., definitive face matches) and slowly expanding as the system's accuracy is proven against the human operator's decisions. This ensures that early mistakes do not break the timeline and undermine user trust.
