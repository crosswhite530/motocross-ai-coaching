# Architecture

This document describes the current deployed architecture and the full target pipeline this project is building toward. They are not the same thing — the Current Status section below is explicit about what's live today versus what's planned.

---

## Current Status

| Component | Status |
|-----------|--------|
| Perspective classification (CLIP, ViT-B/32) | ✅ Deployed — hosted cloud service on Modal |
| Cloud infrastructure (FastAPI, Supabase quota/storage) | ✅ Deployed — validated end-to-end |
| Human detection + pose estimation (YOLOv8-Pose) | ✅ Working locally / 📋 cloud port planned |
| Maneuver classification | ✅ Working locally / 📋 cloud port planned |
| Motorcycle detection | 📋 Planned |
| Claude API coaching integration | 📋 Planned |
| Annotated video output | ✅ Working locally / 📋 next-phase planned |
| Staging environment | 📋 Planned (before next phase of cloud feature work) |

---

## Phase 1: Currently Deployed Architecture

The perspective classification service runs as a three-function Modal app (`motocross-ai-coach`):

```
Client Request (signed Supabase video URL)
    │
    ▼
[fastapi_app / ingest_video] — web endpoint
Checks per-user quota and failure-rate limits.
Returns "accepted" immediately; spawns processing asynchronously.
    │
    ▼
[run_pipeline] — CPU-only Modal function
Downloads video from Supabase Storage.
Validates duration against the 90-second cap.
Normalizes to 720p / 30fps via ffmpeg.
    │
    ▼
[run_inference] — GPU (T4) Modal function
Runs CLIP (ViT-B/32) perspective classifier.
Writes result to Supabase (label, confidence, stability flag).
Increments user quota counter only after confirmed successful write.
```

**Design decisions worth noting** — the full reasoning is in `docs/decision-log.md`, but briefly:

- **CPU/GPU split:** CPU-only validation runs before the GPU function loads. Bad uploads, oversized videos, and network failures fail on cheap hardware before any GPU billing starts.
- **Quota on confirmed success only:** The quota counter increments after the result is successfully written to Supabase — infrastructure failures on the server side don't silently consume a user's monthly submission.
- **Independent failure-rate limiter:** A separate gate (5 failures within 15 minutes) prevents a retry loop of consistently failing requests from running up GPU charges through the gap that success-only quota would otherwise leave open.

---

## Target Architecture: Full Pipeline

This is what the system is building toward. The perspective classifier is the only component currently deployed. Everything downstream of the fork is planned — and the order and structure of that work is not yet decided.

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

For detailed reasoning behind component choices and design decisions at each stage, see `docs/decision-log.md`.
