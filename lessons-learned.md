# Lessons Learned

Things that failed, surprised us, or changed our understanding of the problem. This document is deliberately honest — it shows real engineering problem-solving, not a sanitized success story.

---

## MediaPipe in Uncontrolled Outdoor Environments

**What we tried:** MediaPipe Pose as the primary pose estimation framework.

**What happened:** Import issues prevented it from running. Even before that, initial evaluation suggested it would struggle with motocross footage — it was designed for controlled, close-range environments and doesn't handle the speed, occlusion, and distance variation of outdoor action sports well.

**What we learned:** Framework selection needs to match the *worst case* of your input data, not the average case. Motocross footage is extreme — pick tools built for complex real-world scenes.

---

## 1st Person vs 3rd Person Is a Harder Problem Than Expected

**What we tried:** Training a single model to handle both POV footage and 3rd person camera footage.

**What happened:** The model struggles to distinguish between the two perspectives, leading to incorrect analysis — trying to find body position in POV footage where no rider body is visible, or misclassifying the perspective entirely.

**What we learned:** The visual features between perspectives are so fundamentally different they almost require separate models. A perspective classifier as a preprocessing step is the right architectural move — route the footage before analysis, not during.

---

## Detection Confidence Drops at Distance and Oblique Angles

**What we tried:** Standard YOLOv8-Pose detection on full video frames.

**What happened:** When the rider is small in frame (far from camera) or viewed from an unusual angle, detection confidence drops below threshold and tracking is lost.

**What we learned:** Training data coverage matters enormously. The model is underrepresented on distant and oblique shots. Two approaches to address: (1) more targeted training examples from those scenarios, and (2) adding motorcycle detection as a larger, more consistently visible anchor object.

---

## Off-by-One in a Sampling Gate Can Silently Disable a Component

**What we tried:** CLIP-based PerspectiveClassifier with a per-frame sampling gate (`frame_counter % sample_rate == 1`) to control how often the model runs.

**What happened:** When `sample_rate=1` was passed from the test script, modulo 1 always returns 0 — so the gate condition `== 1` never fired. CLIP ran exactly once on the first frame and returned a cached result on every subsequent call. All 11 sampled frames reported identical confidence (0.6440) with zero perspective switches, making the classifier look broken rather than misconfigured. A second bug compounded the diagnosis: a confidence threshold (`if confidence < 0.65`) was silently overriding CLIP's output with no logging, so even once the gate was fixed, the real model output would have remained hidden.

**What we learned:** Silent overrides and ambiguous gate conditions are a dangerous combination — each one individually produces misleading output, and together they make root cause diagnosis much harder. Two rules going forward: (1) sampling gates should fire on `== 0`, not `== 1`, since modulo with any divisor produces 0 on the first iteration; (2) any branch that overrides a model's output should log that it fired, or it becomes invisible during debugging.

---

## Majority-Vote Smoothing Conflates Noise Robustness and Detection Latency

**What we tried:** A majority-vote window to smooth the raw per-frame perspective classifications — the smoothed label only flips if more than half of the frames in a fixed window agree on the new label.

**What happened:** The smoother worked well enough at suppressing false switches but introduced ~2 seconds of detection lag on real perspective changes. Worse, the actual latency was a function of two coupled parameters (`sample_rate` and `window_size`) with no explicit relationship to real-world time, making it hard to reason about or guarantee how quickly a genuine switch would be detected. The lag only became visible after fixing the sampling/caching bug — the smoother had been masking a problem that would have been obvious in production.

**What we learned:** Smoothing strategies that couple noise robustness and latency in a single parameter are hard to tune and hard to reason about. The better design is to separate the two concerns explicitly: use a consecutive-confidence requirement (N consecutive samples must agree above a confidence threshold) for noise robustness, and derive the sampling rate from video fps to provide a hard real-world latency guarantee. When those are independent levers, you can tighten or loosen either one without breaking the other.

---

*Add new entries as failures and surprises occur. This document is a strength, not a weakness.*
