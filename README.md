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

> This diagram shows the full target architecture. Current implementation status is tracked in the Current Status table below — not all components shown here are deployed yet.

```
Raw Video
    │
    │  Audio track preserved throughout — available to both branches;
    │  how and where it integrates into each is TBD.
    │
    ▼
[Perspective Classifier — CLIP, ViT-B/32]
    │
    ├───────────────────────────────────────────┐
    │                                           │
    ▼                                           ▼
[1ST PERSON / POV branch —          [3RD PERSON branch —
 vision only, not yet decided]       vision only, not yet decided]
TBD: track/line analysis,           TBD: YOLOv8 rider/bike
audio cues, handlebar/              detection, pose extraction,
front-fender visual cues —          maneuver classification,
order and integration               audio cues — order and
undecided                           integration undecided
    │                                           │
    └─────────────────┬─────────────────────────┘
                      ▼
         [Claude API — Coaching Engine]
         Branch-appropriate coaching cues
                      │
                      ▼
         [Annotated Video Output]
         Label + coaching feedback overlay
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Pose estimation | YOLOv8-Pose (Ultralytics) |
| Object detection | YOLOv8 |
| Perspective classification | CLIP (OpenAI, ViT-B/32) |
| Coaching reasoning | Claude API (Anthropic) |
| Video processing | OpenCV |
| Backend / hosting | Modal (serverless CPU + GPU functions), FastAPI |
| Database / storage | Supabase (Postgres + Storage) |
| Runtime | Hybrid — perspective classification is deployed to the cloud (Modal); maneuver detection and coaching are still local, cloud migration planned |

---

## Current Status

| Feature | Status |
|---------|--------|
| 1st/3rd person perspective classifier | ✅ Validated and deployed as a hosted cloud service |
| Cloud deployment & subscription/quota infrastructure | ✅ Working (validated end-to-end) |
| Human detection + pose estimation | ✅ Working locally / 📋 cloud port planned |
| Maneuver classification | ✅ Working locally / 📋 cloud port planned |
| Annotated video output | ✅ Working locally / 📋 next-phase planned |
| Staging environment | 📋 Planned (before next phase of cloud feature work) |
| Motorcycle detection | 📋 Planned |
| Claude API coaching integration | 📋 Planned |
| Distant/oblique angle detection | 🔧 Improving |

---

## Setup

*[Installation and setup instructions will be added here]*

---

## Project Background

This project started from a straightforward question: what would it take to get useful, specific coaching feedback from motocross video? As a rider who has spent years trying to figure out technique largely on my own, the idea of a system that could watch footage and say "your hips were too far back on that jump" rather than just "nice jump" felt worth building — and technically interesting enough to actually hold my attention.

The engineering is genuine: a CLIP-based perspective classifier deployed as a serverless cloud function, a YOLOv8-Pose pipeline for body position analysis, and a coaching feedback layer driven by the Claude API. The architecture decisions — why Modal over a self-hosted GPU, how to structure a subscription and quota system before the feature set is even complete — are documented in the decision log. Not because documentation is the point, but because building something worth showing required actually thinking through those decisions, and writing them down is how I don't lose track of the reasoning.

The longer-term hope is to build something genuinely useful for other riders and coaches — people who want the same kind of specific, technique-focused feedback that's otherwise hard to get outside of a professional training program. A subscription model is the current direction (individual riders, and a separate licensing tier for coaches and training facilities), though the shape of that may evolve as the project develops and I learn more about what the community actually finds valuable. Open-sourcing parts of the pipeline is also something I'm considering down the road — not a current commitment, but genuinely under consideration.

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed component documentation and [docs/decision-log.md](docs/decision-log.md) for engineering decisions made throughout the project.
