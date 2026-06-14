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

*Add new entries as failures and surprises occur. This document is a strength, not a weakness.*
