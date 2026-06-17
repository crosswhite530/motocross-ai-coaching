# Decision Log

This document records key engineering decisions made throughout the project — what was tried, what failed, what was chosen, and why. This is the most important document in the repository for demonstrating engineering thinking.

---

## Format

Each entry follows this structure:

```
## [DATE] — [Decision Title]

**Problem:** What we were trying to solve
**Options considered:** What we evaluated
**Decision:** What we chose
**Reasoning:** Why we chose it
**Outcome:** What happened (updated as results come in)
```

---

## Entries

---

## [May 23, 2026] — Framework Selection: Pose Estimation

**Problem:** Needed a pose estimation framework that could handle motocross footage — fast motion, variable distance, complex environments, no cloud dependency.

**Options considered:**
- **MediaPipe Pose (Google)** — lightweight, real-time, good on body landmarks, lower resource usage. Designed primarily for controlled close-range environments.
- **YOLOv8-Pose (Ultralytics)** — object detection + pose estimation combined, stronger on complex real-world scenes, GPU-accelerated.

**Decision:** YOLOv8-Pose as primary, MediaPipe as fallback to evaluate.

**Reasoning:** Motocross footage is essentially worst-case for MediaPipe — fast motion, partial occlusion, extreme angles, variable distance, outdoor lighting. YOLOv8-Pose is built for complex real-world detection and was more reliable in initial testing on 3rd person footage.

**Outcome:** YOLOv8-Pose confirmed as more reliable. MediaPipe attempted but encountered import issues and deprioritized. Returning to YOLOv8-Pose as stable base.

---

## [May 24, 2026] — Output Format: Annotated Video Overlay

**Problem:** How to present maneuver detection and coaching feedback to the user in a useful, demonstrable way.

**Options considered:**
- Text file / JSON output per frame
- Separate coaching report after processing
- Real-time overlay on output video

**Decision:** Real-time overlay on output video, updating as maneuvers change.

**Reasoning:** Most intuitive for the user — feedback is contextually tied to the exact moment in the footage. Also the most compelling demo artifact for portfolio purposes. A coaching report in isolation loses the visual context.

**Outcome:** Working. Maneuver label updates in real time as footage plays. Next step is adding coaching feedback text alongside the maneuver label.

---

## [June 13, 2026] — 1st Person vs 3rd Person Handling

**Problem:** Model struggles to distinguish POV/helmet cam footage from 3rd person footage. The visual features are fundamentally different between perspectives, causing incorrect analysis when the wrong processing branch runs.

**Options considered:**
- Single model trained on both perspectives
- Preprocessing classifier that routes footage before analysis
- Separate specialized models per perspective

**Decision:** *[In progress — to be finalized]*

**Reasoning under consideration:** 1st person footage has distinctive visual features (different motion blur patterns, horizon behavior, no visible rider body). A lightweight binary classifier (1st/3rd) as a preprocessing step would allow each downstream branch to be optimized for its specific inputs without the single-model approach trying to do too much.

**Outcome:** *[Pending]*

---

## [June 13, 2026] — Motorcycle Detection Addition

**Problem:** When rider is far from camera or at oblique angles, human/pose detection confidence drops below threshold and detection is lost.

**Reasoning for planned addition:** The motorcycle is a larger, more consistently visible target than the human body at distance. Adding bike detection creates a secondary anchor for the frame — when human detection fails, bike position still gives spatial context. Additionally, bike geometry (lean angle, suspension compression, wheel alignment relative to jump face) carries independent coaching value that human pose alone cannot provide.

**Status:** Planned for next development phase.

---

## [June 17, 2026] — PerspectiveClassifier: Sampling Gate and Confidence Override Bug Fix

**Problem:** Validation testing of the CLIP-based PerspectiveClassifier (see June 13, 2026 — "1st Person vs 3rd Person Handling") returned identical confidence scores (0.6440) across all 11 sampled frames with zero perspective switches detected, despite the test video containing both 3rd person and POV footage. The classifier appeared functional but was producing useless output.

**Options considered:**
- **Model quality failure** — CLIP embeddings may not separate 1st/3rd person perspectives well on motocross footage, requiring a different approach entirely
- **Sampling gate logic bug** — the `frame_counter % sample_rate` condition may not be firing correctly, causing CLIP to run only once and return a cached result on every subsequent call
- **Silent output override** — a confidence threshold branch may be overriding the model's real predictions without logging, masking the actual classification result

**Decision:** Trace the logic path before concluding model failure. Both logic bugs confirmed and fixed: gate condition changed from `== 1` to `== 0`, `frame_counter` increment moved to after the gate check, and confidence threshold override removed pending further validation on more footage.

**Reasoning:** Identical confidence values across all frames was a strong signal that CLIP was not running per-frame — a caching or gate issue, not a model quality issue. Diagnosing the execution path first avoided prematurely abandoning CLIP. The confidence override was particularly deceptive: it produced plausible-looking output while hiding what the model actually predicted, with no log entry to indicate it had fired.

**Outcome:** Validation passed after the fix. Confidence values now vary per frame (range: 0.6440–0.9307). Classifier correctly detected 1 perspective switch at frame 150 (THIRD_PERSON → FIRST_PERSON) with high confidence on both sides (0.93 and 0.85), consistent with known test video content. Resolves the open decision from June 13 — CLIP-based binary classification is confirmed viable. Next step: stress test on pure single-perspective footage before pipeline integration.

---

*Add new entries above this line as decisions are made.*
