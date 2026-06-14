# Motocross AI Coaching System

> An end-to-end computer vision pipeline that analyzes motocross footage, identifies maneuvers in real time, and provides specific, actionable coaching feedback as a video overlay.

---

## Demo

*[Demo video will be added here when system reaches milestone 1 completion]*

---

## What It Does

This system takes raw motocross video as input and outputs an annotated video with:

- **Maneuver detection** — real-time label that updates as the rider's maneuver changes (jump, whip, scrub, corner, whoops, etc.)
- **Body position analysis** — pose keypoints tracked across frames
- **Coaching feedback** — specific improvement cues generated from biomechanical analysis (e.g., *"Lean 5° farther left on takeoff for a more effective whip"*)

The system handles both **1st person (helmet/POV cam)** and **3rd person** footage through a perspective classification preprocessing step.

---

## Architecture

```
Raw Video
    │
    ▼
[Perspective Classifier]
1st person → POV analysis branch (line selection, track reading)
3rd person → Full biomechanical analysis branch
    │
    ▼
[YOLOv8 — Rider + Bike Detection]
    │
    ▼
[YOLOv8-Pose — Keypoint Extraction]
Shoulders, elbows, hips, knees, ankles
    │
    ▼
[Maneuver Classifier]
Jump / Whip / Scrub / Corner / Whoops / etc.
    │
    ▼
[Claude API — Coaching Feedback Engine]
Keypoints + maneuver type → biomechanical reasoning → coaching cue
    │
    ▼
[Annotated Video Output]
Maneuver label + coaching feedback overlay
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Pose estimation | YOLOv8-Pose (Ultralytics) |
| Object detection | YOLOv8 |
| Coaching reasoning | Claude API (Anthropic) |
| Video processing | OpenCV |
| Runtime | Local / offline (no cloud inference costs) |

---

## Current Status

| Feature | Status |
|---------|--------|
| Human detection + pose estimation | ✅ Working |
| Maneuver classification | ✅ Working |
| Annotated video output | ✅ Working |
| 1st/3rd person classifier | 🔧 In progress |
| Motorcycle detection | 📋 Planned |
| Claude API coaching integration | 📋 Planned |
| Distant/oblique angle detection | 🔧 Improving |

---

## Setup

*[Installation and setup instructions will be added here]*

---

## Project Background

This project was built to demonstrate real-world AI engineering — designing, iterating, and evaluating a computer vision pipeline applied to a domain-specific problem. It is part of a broader portfolio targeting AI engineering roles.

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed component documentation and [docs/decision-log.md](docs/decision-log.md) for engineering decisions made throughout the project.
