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

**Decision:** CLIP zero-shot binary classifier as a preprocessing step — routes footage by perspective before any downstream analysis runs. See June 17, 2026 entries for full implementation detail, bug-fix history, and smoothing strategy.

**Reasoning under consideration:** 1st person footage has distinctive visual features (different motion blur patterns, horizon behavior, no visible rider body). A lightweight binary classifier (1st/3rd) as a preprocessing step would allow each downstream branch to be optimized for its specific inputs without the single-model approach trying to do too much.

**Outcome:** Resolved. CLIP-based classification validated June 17, 2026 (see "PerspectiveClassifier: Sampling Gate and Confidence Override Bug Fix" and "PerspectiveClassifier: Smoothing Strategy and Latency Guarantee"). Deployed as a hosted service June 23, 2026 and validated end-to-end with real footage — see "Deploying the Perspective Classifier as a Hosted Service."

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

## [June 17, 2026] — PerspectiveClassifier: Smoothing Strategy and Latency Guarantee

**Problem:** After fixing the sampling/caching bug (see June 17, 2026 — "PerspectiveClassifier: Sampling Gate and Confidence Override Bug Fix"), validation revealed that the majority-vote smoothing window introduced ~2 seconds of detection lag on real perspective changes. Additionally, `sample_rate` and `window_size` were coupled in a way that made actual detection latency hard to reason about or guarantee — the relationship between those two parameters and the real-world time-to-detect was not explicit.

**Options considered:**
- **Keep majority-vote smoothing, tune window size** — reducing the window size would lower latency but increase false switch rate; the tradeoff is not well-controlled and the latency is still a function of two coupled parameters
- **Consecutive-confidence method** — flip the smoothed perspective only when the last N consecutive raw samples all agree on the same label AND each meets a minimum confidence threshold; derive `sample_rate` from the video's actual fps so that N consecutive samples always fit within a fixed real-world latency target

**Decision:** Consecutive-confidence method with `required_consecutive=3`, `min_confidence=0.65`, and `sample_rate` derived from video fps as `(target_latency_sec * fps) // required_consecutive`.

**Reasoning:** The consecutive-confidence approach decouples the two concerns that majority-vote conflated: noise robustness (controlled by `required_consecutive` and `min_confidence`) and latency guarantee (controlled by the `sample_rate` derivation). Holding `required_consecutive` fixed at 3 regardless of fps keeps CLIP calls per second constant across different frame rate videos — scaling it up with fps would have increased compute cost for a noise-robustness benefit not yet proven necessary. Deriving `sample_rate` from fps means the same 0.5 second latency target is preserved across videos without manual tuning per file.

**Outcome:** Validated on two 30fps videos (derived `sample_rate=5`). Pure 3rd-person footage (My_Whip_Attempt.mp4): 0 false switches across the full clip, down from 4 with majority-vote smoothing. Mixed footage with a real perspective change (Motocross Whip.mp4): switch detected with 0.33s latency (raw label changed at 5.00s, smoothed output confirmed at 5.33s) against a 0.5s target — major improvement over the ~2s lag of the prior approach. Post-switch low-confidence flips back to the prior label were correctly ignored. Files modified: `Python Files/perspective_classifier.py` (added `set_fps()`, `smooth_perspective_consecutive()`; kept legacy `smooth_perspective()` for reference), `Python Files/test_perspective.py` (reads fps via `cv2.CAP_PROP_FPS` instead of assuming 30fps).

---

## [June 17, 2026] — PerspectiveClassifier: Deferring YOLO-Based Detection Heuristic

**Problem:** After fixing the sampling bug and improving smoothing logic (see June 17, 2026 entries above), we considered adding a second, independent perspective signal: using the existing YOLOv8 rider/bike detector to produce perspective cues — full rider and bike visible in frame suggesting third-person, handlebars or hands visible at the bottom of frame suggesting first-person/POV. This was originally motivated by a noisy, low-confidence stretch (0.50–0.65 confidence, frequent label flips) observed in early testing on My_Whip_Attempt.mp4, before the consecutive-confidence smoothing fix was implemented.

**Options considered:**
- **Build the YOLO-based heuristic now** — run it independently from CLIP, flag disagreements between the two methods, accumulate data for future fusion or arbitration logic
- **Defer and re-evaluate** — revisit only if the classifier demonstrates accuracy problems on additional footage going forward

**Decision:** Defer the YOLO-based detection heuristic. The problem that motivated it no longer has clear evidence behind it.

**Reasoning:** The consecutive-confidence smoothing fix already resolved the noisy-stretch issue that prompted this idea — validated at 0 false switches on pure third-person footage and 0.33s detection latency on mixed footage against a 0.5s target. Adding a second model signal, a disagreement-flagging mechanism, and eventual fusion logic before there is a demonstrated need adds engineering overhead and new surface area for bugs without a corresponding, currently-justified benefit. The idea is documented here and available to revisit if the classifier shows accuracy problems on footage types not yet tested (e.g. very low light, extreme oblique angles, very distant riders).

**Outcome:** Moving on to other pipeline priorities. This entry records that the YOLO-based perspective heuristic was deliberately considered and deferred — not overlooked — and documents the conditions under which it should be revisited.

---

## [June 23, 2026] — Deploying the Perspective Classifier as a Hosted Service

**Problem:** The CLIP-based perspective classifier (see June 17, 2026 entries) was validated locally, but a local script can't serve requests from a mobile app — it only runs when a developer's machine is on, and it can't scale to multiple users. The validated logic needed to become a hosted service.

**Options considered:**

- **Monolithic GPU function vs. split CPU/GPU functions:** One option was a single cloud function handling video download, format normalization, and GPU classification all in one place. The alternative was splitting CPU-only preprocessing (download, validate, normalize) from GPU-only classification into separate functions. The key consideration: cloud GPU instances are significantly more expensive per second than CPU instances, and most failure modes (corrupt uploads, oversized files, network errors) are detectable before any GPU work is needed.

- **Quota consumption at request acceptance vs. on confirmed success only:** The simpler design deducts from a user's monthly quota the moment a valid request arrives. The alternative deducts only after a successful result is written to the database. The question is who absorbs the cost of infrastructure failures — the user's submission count, or the operator's risk exposure.

- **Single generic error vs. structured per-checkpoint error codes:** Whether to return a uniform "something went wrong" message or distinct, actionable error codes at each failure point: upload unreachable, invalid video format, duration limit exceeded, normalization failure, classification failure.

- **Flat 90-second duration cap vs. tiered or pay-per-length caps:** Whether to vary the video duration limit by subscription tier or video length, or use a single flat cap for all users. The decision depends on knowing the cost structure of the full pipeline — including Phase 2 components not yet built.

**Decision:** Three-function Modal architecture:

- **`fastapi_app` / `ingest_video`** (web endpoint): Checks quota and failure-rate limits, spawns processing asynchronously, and returns an immediate "accepted" response to the client. Keeps the user-facing request fast and decoupled from processing time.
- **`run_pipeline`** (CPU): Downloads the video from cloud storage, validates duration against the 90-second cap, and normalizes to 720p/30fps via ffmpeg. All work that doesn't require a GPU happens here.
- **`run_inference`** (GPU T4): Runs the CLIP perspective classifier on the normalized video, writes the result to the database, and only then increments the user's quota counter. Classification happens here and only here.

Quota is decremented on confirmed success only. A separate failure-rate limiter — 5 failures within any 15-minute window, tracked independently of the quota counter — gates continued access after repeated failures. Structured error codes surface distinct messages at each failure checkpoint.

Duration cap: flat 90 seconds across all tiers for now.

**Reasoning:** The CPU/GPU split is the central architectural decision and the reasoning is straightforward: cheap CPU infrastructure should absorb the work that doesn't require expensive GPU resources. Corrupt uploads, oversized videos, and network failures are all detectable during preprocessing — catching them there means they never reach the GPU billing clock.

Gating quota on confirmed success rather than request acceptance means infrastructure failures — a transient cloud error, a processing bug, a GPU timeout — don't silently consume a user's monthly submission. Users should only be charged for what actually worked.

The failure-rate limiter closes a cost-exposure gap the success-only quota design would otherwise leave open: a client retrying failed requests in a tight loop wouldn't consume quota under the success-only model, but it could still repeatedly spin up GPU instances. The two mechanisms are independent by design — one protects the user from infrastructure failures, the other protects the service from retry-loop cost exposure.

Keeping the duration cap flat defers a decision that depends on cost data that doesn't exist yet. Phase 2 (maneuver detection and coaching) will substantially change the cost profile of processing a 60-second video vs. a 30-second one. Committing to a tiered cap structure before those costs are measured would be guessing at the wrong time.

**Outcome:** Deployed to Modal on June 23, 2026. First full end-to-end validation completed the same day using a real third-person test clip: the pipeline returned `label: "third_person"`, `confidence: 0.9033`, `is_stable: true`, and the user's quota incremented from 0 to 1 only after the successful result write — confirming the full chain: request accepted → quota checked → processing spawned asynchronously → video downloaded and normalized → GPU classification run → result written to database → quota incremented on success.

---

## [June 23, 2026] — Staging Infrastructure Before Phase 2 Feature Work

**Problem:** Every deployment to this point went directly to the production Modal app and Supabase project. While nothing real was running, that was acceptable. With the perspective classifier now validated and serving correctly in production, adding new pipeline components (the next step being YOLOv8-Pose detection) by redeploying directly to the production environment risks disrupting a pipeline that currently works.

**Options considered:**
- Continue deploying directly to production and accept the disruption risk as new components are added incrementally.
- Set up an isolated staging environment before writing any Phase 2 pipeline code, and validate all new components in staging before promoting to production.

**Decision:** Set up a dedicated Modal staging environment (separate from the production `main` environment) and a separate free-tier Supabase project for staging — fully isolated from production data and tables. All Phase 2 components get built and validated in staging first; production only receives changes confirmed working.

**Reasoning:** The perspective classifier pipeline now works correctly in production. Treating it as the stable baseline worth protecting — rather than a convenient place to keep iterating — is the right shift in posture at this point in the project. Staging isolation means a broken YOLOv8 integration or a schema change can't take down a working service. It also keeps test data out of production tables that will need to stay trustworthy once real users exist.

**Outcome:** Implemented and validated on June 27, 2026. A dedicated Modal staging environment and a separate Supabase project were created, each fully isolated from production. The existing service code deployed to staging without modification — Modal's environment-scoped secrets mean the same `app.py` picks up staging credentials automatically. Validated end-to-end with a real video: pipeline classified correctly, quota incremented on confirmed success only, and production data confirmed unaffected by the staging test run.

---

*Add new entries above this line as decisions are made.*
