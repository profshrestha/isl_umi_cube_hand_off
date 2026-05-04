# Step 2 — SLAM Pipeline

The SLAM pipeline processes your raw GoPro videos into `replay_buffer.zarr.zip` — the training dataset. It runs ORB-SLAM3 to localize each camera, detects ArUco tags for gripper width, and packages everything into a zarr file.

**Run this on the lab workstation** (Ubuntu 22.04, UMI conda environment at `/opt/miniconda3/envs/umi`, UMI repo at `/opt/umi/universal_manipulation_interface/`).

---

## Prerequisites

The lab workstation already has everything installed:
- UMI conda environment (`umi`)
- ORB-SLAM3 (runs via Docker internally in the pipeline)
- OpenCV 4.11 with the ArUco fix applied

You do not need to install anything.

---

## Step 2.1 — Prepare Your Session Directory

Your session directory must be organized before running the pipeline.
Transfer your videos to the workstation and arrange them as follows:

```
file_path/datasets/<your_session>/
└── raw_videos/
    ├── mapping.mp4
    ├── gripper_calibration/
    │   ├── left_calib.mp4
    │   └── right_calib.mp4
    └── (all demo .mp4 files — any filenames are fine)
```

**Important:**
- The mapping video must be named exactly `mapping.mp4` (lowercase).
- Calibration clips go inside `gripper_calibration/`. The pipeline uses all clips in that folder.
- Demo clips can have any name — they are paired by timestamp, not filename.

---

## Step 2.2 — Run the SLAM Pipeline

Activate the conda environment and run:

```bash
conda run -n umi --no-capture-output \
  python /opt/umi/universal_manipulation_interface/run_slam_pipeline.py \
  file_path/datasets/<your_session>/
```

This runs steps 00–07 automatically. The full pipeline takes **1–2 hours** depending on the number of clips.

### Output

On success, the pipeline writes:

```
<your_session>/replay_buffer.zarr.zip    # training dataset (~200–500 MB)
<your_session>/dataset_plan.pkl          # which demo pairs were used
<your_session>/demos/                    # intermediate SLAM outputs
```

The final line printed will be:
```
Pipeline complete. Replay buffer saved to: <your_session>/replay_buffer.zarr.zip
```

---

## Optional — Run SLAM on the ISL Server

If you do not have the UMI environment set up locally, you can run the full pipeline on the ISL server where the raw videos are already available.

> **Only one person should run the pipeline at a time.** If a classmate has already run it, skip to the `scp` step below — the zarr is already there.

### 1 — SSH in

```bash
ssh <your_username>@130.157.158.135
```

### 2 — Run the pipeline

Start a `tmux` session so the job survives if your connection drops:

```bash
tmux new -s pipeline

/opt/miniconda3/envs/umi/bin/python \
  /opt/umi/universal_manipulation_interface/run_slam_pipeline.py \
  /var/isl_robotics_shared/isl_umi/datasets/cube_hand_off/session_may326

# Detach and leave it running: Ctrl+B, then D
# Reattach later:
tmux attach -t pipeline
```

Takes **2–4 hours**. When done you will see:
```
Pipeline complete. Replay buffer saved to: .../session_may326/replay_buffer.zarr.zip
```

### 3 — Copy the zarr to your machine

Run this **on your local machine**:

```bash
scp <your_username>@130.157.158.135:/var/isl_robotics_shared/isl_umi/datasets/cube_hand_off/session_may326/replay_buffer.zarr.zip ~/Downloads/
```

The file is ~2–3 GB. Then continue with Step 2.3 below.

---

## Step 2.3 — Check How Many Demos Were Recovered

```bash
python - <<'EOF'
import zarr, zipfile
with zipfile.ZipFile("file_path/datasets/<your_session>/replay_buffer.zarr.zip") as zf:
    store = zarr.ZipStore("file_path/datasets/<your_session>/replay_buffer.zarr.zip", mode='r')
    root = zarr.open(store, 'r')
    print(f"Episodes: {len(root['meta/episode_ends'])}")
    print(f"Total frames: {root['meta/episode_ends'][-1]}")
EOF
```

**Target: at least 50 paired demos.** If you recover fewer than 30, the policy will likely underfit. Common causes:

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| < 30% of clips localized | Mapping video too short or incomplete | Re-record mapping video |
| Many clips dropped in step 06 | `nominal_z` wrong | See Troubleshooting below |
| Clips localize but pairs don't match | Cameras not time-synced | Re-sync before next session |

---

## Step 2.4 — Upload the Zarr to NRP

Once you have a good zarr file, upload it to the NRP shared workspace so training can use it.

### Launch the staging pod

```bash
kubectl apply -f folder_path/isl_umi_cube_hand_off/nrp/pvc_stage_pod.yaml
kubectl wait --for=condition=Ready pod/pvc-stage -n ssu-intelligent-systems --timeout=120s
```

### Upload

```bash
kubectl cp \
  file_path/datasets/<your_session>/replay_buffer.zarr.zip \
  ssu-intelligent-systems/pvc-stage:/workspace/data/<your_session>.zarr.zip
```

### Verify

```bash
kubectl exec -n ssu-intelligent-systems pvc-stage -- ls -lh /workspace/data/
```

### Clean up

```bash
kubectl delete pod pvc-stage -n ssu-intelligent-systems
```

Then follow **[Step 3 — NRP Training](03_nrp_training.md)**.

---

## Troubleshooting

### Step 02 fails (SLAM map not built)

Check that `mapping.mp4` exists and is a valid video:
```bash
ffprobe file_path/datasets/<your_session>/raw_videos/mapping.mp4
```
If the mapping step fails, the rest of the pipeline cannot run. Re-record the mapping video.

### Step 04 completes but detects no tags

Verify the ArUco tags are visible in your videos:
```bash
ls file_path/datasets/<your_session>/demos/demo_*/tag_detection.pkl
```
Each file should be non-empty. If all are empty/missing, there is a camera intrinsics or ArUco config mismatch — ask Prof. Shrestha.

### Step 05 or 06 crash with NaN or empty list

This usually means the `nominal_z` filter is rejecting all measurements. The default is 0.072 m (7.2 cm between gripper fingers). If your gripper was held closer during calibration, you need to pass a lower value:

```bash
conda run -n umi --no-capture-output \
  python /opt/umi/universal_manipulation_interface/run_slam_pipeline.py \
  file_path/datasets/<your_session>/ \
  --nominal_z 0.050
```

Measure the actual finger-tag distance from your calibration video and set accordingly. If unsure, start with `0.050` and adjust.

### Very few demos recovered (< 20%)

SLAM localization can fail if the scene changes between mapping and demos (lighting, objects moved) or if the camera motion during a demo is too fast. 

Check which cameras failed:
```bash
ls file_path/datasets/<your_session>/demos/*/camera_trajectory.csv | wc -l
```
Divide by the total number of demo clips. If one camera has a much higher failure rate than the other, its mapping coverage was insufficient.

Re-record the mapping video covering the scene from that camera's typical viewpoint.
