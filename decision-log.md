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

*Add new entries above this line as decisions are made.*
