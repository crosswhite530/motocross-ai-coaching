# Progress Journal

Weekly notes on what changed, what's working, what's blocking, and what's next.

---

## Format

```
## Week of [DATE]

**What changed this week:**
**What's working now that wasn't:**
**What's blocking me:**
**Next session focus:**
```

---

## Entries

---

## Week of [May 23, 2026]

**What changed this week:**
- Established core pipeline: video input → YOLOv8-Pose → maneuver classification → annotated video output
- Human detection and body position identification working
- Maneuver detection working for: jumps, corners, whoops, whips
- Output video with real-time maneuver label overlay working

**What's working now that wasn't:**
- End-to-end pipeline producing annotated output video
- Maneuver label updates dynamically as rider's action changes in frame

**What's blocking me:**
- 1st person vs 3rd person footage not distinguished reliably
- Detection drops when rider is distant or at unusual angles
- MediaPipe import issues (deprioritized)

**Next session focus:**
- Re-stabilize YOLOv8-Pose as clean working base
- Begin 1st/3rd person classifier design

---

## Week of [June 17, 2026]

**What changed this week:**
- Diagnosed and fixed two bugs in `perspective_classifier.py`: sampling gate (`frame_counter % sample_rate == 1`) never fired when `sample_rate=1` because modulo 1 always returns 0; a confidence threshold branch was silently overriding CLIP's output with no logging
- Validation re-run confirmed both fixes: classifier now runs per-frame and produces real varying output

**What's working now that wasn't:**
- CLIP-based PerspectiveClassifier producing real per-frame classifications instead of a single cached result repeated across all frames
- Perspective switch detection working on mixed footage: 1 switch detected at frame 150 (THIRD_PERSON → FIRST_PERSON), confidence range 0.6440–0.9307, consistent with test video content

**What's blocking me:**
- Classifier not yet validated on pure single-perspective footage — need to confirm no false switches before trusting it in the main pipeline

**Next session focus:**
- Stress test on pure 3rd-person-only and pure 1st-person-only footage, then integrate into main pipeline if results hold

---

## Week of [June 17, 2026] — Smoothing and Latency Improvement

**What changed this week:**
- Replaced majority-vote smoothing in `PerspectiveClassifier` with a consecutive-confidence method: smoothed perspective now only flips when the last 3 consecutive raw samples agree on the same label and each meets a 0.65 confidence threshold
- `sample_rate` is now derived from the video's actual fps (`(target_latency_sec * fps) // required_consecutive`) so the 0.5s latency target holds across different frame rates without manual tuning
- `set_fps()` and `smooth_perspective_consecutive()` added to `perspective_classifier.py`; legacy `smooth_perspective()` kept for reference
- `test_perspective.py` updated to read fps via `cv2.CAP_PROP_FPS` instead of assuming 30fps; now logs both frame number and elapsed seconds

**What's working now that wasn't:**
- False switch rate on pure 3rd-person footage dropped from 4 to 0 (My_Whip_Attempt.mp4, full clip)
- Real perspective switch detected in 0.33s (Motocross Whip.mp4, raw change at 5.00s, smoothed confirmed at 5.33s) — down from ~2s lag with majority-vote approach
- Post-switch low-confidence raw flips correctly ignored by smoothed output on both sides of the transition

**What's blocking me:**
- Only validated at 30fps; behavior at other frame rates not yet tested

**Next session focus:**
- Build a YOLO-based detection heuristic as an independent second signal alongside CLIP: bike/rider fully visible → 3rd person cue; handlebars/hands at bottom of frame → 1st person cue; compare agreement/disagreement between the two before considering whether to combine them

---

*Add new entries above this line each week.*
