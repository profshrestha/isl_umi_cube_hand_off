# Step 1 — Data Collection

## Hardware Setup

Follow the official UMI assembly and calibration guide for hardware setup, GoPro mounting, and gripper construction:

> **https://umi-gripper.github.io**

The lab grippers are already assembled. Come to Prof. Shrestha before your recording session to verify the ArUco tags are correctly attached (see below).

---

## ArUco Tag IDs — Critical

The SLAM pipeline assigns gripper calibrations by tag ID. **Both grippers must use different IDs:**

| Gripper | Tags |
|---------|------|
| Left arm | IDs **0** and **1** |
| Right arm | IDs **6** and **7** |

Using the same IDs on both grippers (e.g., 0 and 1 on both) requires a pipeline workaround and will cause calibration issues. Verify before every session.

---

## Session Directory Layout

Create a session folder before you start. Everything goes inside it:

```
my_session/
├── raw_videos/
│   ├── mapping.mp4              # One mapping video (see below)
│   ├── gripper_calibration/     # 2 calibration clips (one per camera)
│   └── demo_001_left.mp4        # Demo clips — one per camera per take
│   └── demo_001_right.mp4
│   └── demo_002_left.mp4
│   └── ...
```

The SLAM pipeline pairs demo clips by **timestamp overlap** — not by filename. You do not need to name them in any particular order, but keeping a consistent naming convention helps you stay organized.

---

## Time Synchronization

**Sync both GoPros before every session.** The demo pairing step relies on timestamps overlapping. If the cameras are out of sync by more than a few seconds, demos will fail to pair and be dropped.

Use GoPro's built-in time sync or perform a visual sync cue: hold both cameras pointed at the same clock or flash an LED in front of both cameras at the start of the session.

---

## Recording the Mapping Video

The mapping video is used by ORB-SLAM3 to build a map of the environment. A poor mapping video is the single biggest cause of SLAM localization failure.

- Record **one** mapping video per session (one camera is enough — use either).
- Walk **slowly** through the full workspace.
- Cover the workspace from **multiple angles and heights**, including close-up views of the table surface where demos will happen.
- Record in the **same lighting conditions** as the demos — do not change the lights between mapping and demos.
- Aim for at least 2–3 minutes of footage.

---

## Gripper Calibration Clips

Record one calibration clip per camera:

- Mount the camera as it will be during demos.
- Slowly **open and close** the gripper several times in front of the camera.
- The finger tags should be clearly visible throughout.
- Keep the gripper at roughly its **normal operating distance** from the camera (~7 cm between fingers is ideal).

Note the approximate open/close distance — you will need it as `nominal_z` in the SLAM pipeline.

---

## Recording Demos

**Target: 100 successful demonstrations.**

Given SLAM localization failures (~30–50% of clips may fail), record **150–200 raw clips** to recover ~100 after filtering.

### Task
Pick up the small cube and place it into the cardboard box using both arms cooperatively.

### Location Variation
Vary the pickup position and box position **continuously and randomly** within the reachable workspace zone. Do not keep them fixed — a policy trained on a single location will fail if the object moves even slightly.

A practical approach: after placing the box, shift it a few centimeters in a random direction. After each demo, move the cube starting position slightly.

### Checklist per Demo
- [ ] Both cameras recording before the arms start moving
- [ ] The cube is placed at a slightly different location from the previous demo
- [ ] The box is placed at a slightly different location from the previous demo
- [ ] Both arms return to a neutral pose before stopping recording
- [ ] Demo is clean (no collisions, drops, or restarts mid-clip)

### What to Avoid
- Starting or stopping recording after the task begins
- Occluding the wrist camera with your hand during the demo
- Changing lighting mid-session
- Moving the cameras between mapping and demos
- Recording demos in a location not covered by the mapping video

---

## After Recording

Transfer all videos to the lab workstation:

```
/home/shrestha/sudhir_umi/datasets/<your_session_name>/raw_videos/
```

Then follow **[Step 2 — SLAM Pipeline](02_slam_pipeline.md)**.
