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

*Add new entries above this line each week.*
