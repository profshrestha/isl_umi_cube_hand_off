# ISL Robotics — UMI Cube Hand-Off Project

This repository contains everything students need to collect demonstrations, run the SLAM pipeline, train a bimanual diffusion policy on NRP, and download the resulting checkpoints.

**Task:** Pick up a small cube and place it in a cardboard box using two RX150 robot arms (bimanual).

---

## Pipeline Overview

```
1. Data Collection       2. SLAM Pipeline          3. NRP Training          4. Get Checkpoints
  (GoPro + grippers)  →  (local workstation)   →   (Kubernetes cluster)  →  (download .ckpt)
  docs/01_data_           docs/02_slam_             docs/03_nrp_             docs/04_checkpoints
  collection.md           pipeline.md               training.md              .md
```

---

## Quick Links

| Step | Guide | Est. Time |
|------|-------|-----------|
| 1. Collect demos | [Data Collection](docs/01_data_collection.md) | ~2–3 h per 100-demo session |
| 2. Run SLAM | [SLAM Pipeline](docs/02_slam_pipeline.md) | ~1–2 h on lab workstation |
| 3. Train on NRP | [NRP Training](docs/03_nrp_training.md) | ~5–7 days on cluster |
| 4. Download checkpoints | [Checkpoints](docs/04_checkpoints.md) | ~5 min |

---

## Lab Hardware

| Item | Detail |
|------|--------|
| Robot arms | 2× Interbotix RX150 |
| Cameras | 2× GoPro (wrist-mounted, one per gripper) |
| Camera serials | Left: C3441324937748 · Right: C3444250515560 |
| ArUco tags | Left gripper: IDs **0, 1** · Right gripper: IDs **6, 7** |
| NRP namespace | `ssu-intelligent-systems` |
| PVC | `umi-workspace-pvc` (100 Gi, shared) |
| Docker image | `profshrestha/umi-rx150:latest` |

> **ArUco tag IDs are critical.** The SLAM pipeline uses tag IDs to assign gripper calibrations. Left and right grippers must use different IDs (0,1 vs 6,7). Using the same IDs on both grippers will corrupt the calibration lookup.

---

## Repository Contents

```
isl_umi_cube_hand_off/
├── README.md
├── docs/
│   ├── 01_data_collection.md   # Recording demos — what to do and what to avoid
│   ├── 02_slam_pipeline.md     # Processing videos into a training dataset (zarr)
│   ├── 03_nrp_training.md      # Submitting and monitoring training on NRP
│   └── 04_checkpoints.md       # Downloading trained model checkpoints
└── nrp/
    ├── umi_train_cube_job.yaml  # Kubernetes training job — edit name/paths per run
    └── pvc_stage_pod.yaml       # Staging pod for uploading data to the PVC
```

---

## Getting Help

- UMI project website and hardware assembly: https://umi-gripper.github.io
- NRP (Nautilus) documentation: https://docs.nationalresearchplatform.org
- Lab contact: Prof. Shrestha
