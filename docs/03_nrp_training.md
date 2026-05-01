# Step 3 — NRP Training

Training runs on the National Research Platform (NRP) Kubernetes cluster using a shared GPU node. You submit a Job, the cluster assigns a GPU, and training runs until completion.

---

## Prerequisites

### kubectl

Install kubectl: https://kubernetes.io/docs/tasks/tools/

Configure it with the NRP kubeconfig:
1. Log in to the NRP portal (you should already have access via Prof. Shrestha).
2. Download your kubeconfig file from the portal.
3. Save it to `~/.kube/config`.

Verify:
```bash
kubectl get pods -n ssu-intelligent-systems
```
You should see any currently running pods without errors.

---

## Step 3.1 — Upload Your Dataset to the PVC

The training job reads from the shared PVC (`umi-workspace-pvc`). Upload your zarr file before submitting the job.

```bash
# Launch staging pod
kubectl apply -f nrp/pvc_stage_pod.yaml
kubectl wait --for=condition=Ready pod/pvc-stage -n ssu-intelligent-systems --timeout=120s

# Upload your zarr (replace filename as needed)
kubectl cp \
  /path/to/your/replay_buffer.zarr.zip \
  ssu-intelligent-systems/pvc-stage:/workspace/data/<your_session>.zarr.zip

# Confirm it's there
kubectl exec -n ssu-intelligent-systems pvc-stage -- ls -lh /workspace/data/

# Clean up staging pod
kubectl delete pod pvc-stage -n ssu-intelligent-systems
```

---

## Step 3.2 — Configure the Job YAML

Copy the template and edit it for your run:

```bash
cp nrp/umi_train_cube_job.yaml nrp/umi_train_<your_name>_v1.yaml
```

Edit these three lines in your copy:

```yaml
metadata:
  name: umi-train-<your_name>-v1        # must be unique in the namespace
```

```bash
              DATASET=/workspace/data/<your_session>.zarr.zip   # your uploaded file
```

```bash
              mkdir -p /workspace/checkpoints/<your_name>_001
              ...
              hydra.run.dir=/workspace/checkpoints/<your_name>_001
```

> **Job names must be unique.** If a job with the same name already exists (even finished), delete it first:
> ```bash
> kubectl delete job umi-train-<your_name>-v1 -n ssu-intelligent-systems
> ```

---

## Step 3.3 — Submit the Job

```bash
kubectl apply -f nrp/umi_train_<your_name>_v1.yaml
```

Check that it scheduled:
```bash
kubectl get pods -n ssu-intelligent-systems -l job-name=umi-train-<your_name>-v1
```

The pod will show `Pending` briefly while the cluster finds a free GPU node, then switch to `Running`. If it stays `Pending` for more than 15 minutes, see Troubleshooting.

---

## Step 3.4 — Monitor Training

Get your pod name:
```bash
POD=$(kubectl get pods -n ssu-intelligent-systems \
      -l job-name=umi-train-<your_name>-v1 \
      -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Stream the log live:
```bash
kubectl exec -n ssu-intelligent-systems $POD -- \
  tail -f /workspace/logs/cube_train.log
```

Check progress (recent lines only):
```bash
kubectl exec -n ssu-intelligent-systems $POD -- \
  bash -c "tail -c 3000 /workspace/logs/cube_train.log | strings | tail -20"
```

List checkpoints saved so far:
```bash
kubectl exec -n ssu-intelligent-systems $POD -- \
  ls /workspace/checkpoints/<your_name>_001/checkpoints/
```

Checkpoints are saved every 10 epochs. The log shows lines like:
```
Training epoch 10: 100%|████████| 72/72 [02:15<00:00, loss=0.014]
```

**Expected training time:** 5–7 days for 100 epochs on a typical NRP GPU.

---

## Step 3.5 — After Training Completes

When training finishes the pod will show `Completed`. Proceed to **[Step 4 — Download Checkpoints](04_checkpoints.md)**.

---

## Troubleshooting

### Pod stays Pending

The cluster may be busy. Check what's happening:
```bash
kubectl describe pod -n ssu-intelligent-systems $POD
```
Scroll to the `Events` section. `Insufficient nvidia.com/gpu` means all GPU nodes are occupied — wait and it will schedule automatically.

### OOMKilled (GPU out of memory)

The pod will show `OOMKilled` in `kubectl get pods`. Edit your YAML and halve the batch sizes:

```yaml
dataloader.batch_size=4 \
val_dataloader.batch_size=1 \
```

Delete the failed job and resubmit with `v2`:
```bash
kubectl delete job umi-train-<your_name>-v1 -n ssu-intelligent-systems
# edit YAML: rename to v2, reduce batch sizes
kubectl apply -f nrp/umi_train_<your_name>_v2.yaml
```

### Dataset not found

The job will exit immediately with `ERROR: dataset not found`. Launch the staging pod and verify the file is at the expected path on the PVC:
```bash
kubectl apply -f nrp/pvc_stage_pod.yaml
kubectl exec -n ssu-intelligent-systems pvc-stage -- ls -lh /workspace/data/
kubectl delete pod pvc-stage -n ssu-intelligent-systems
```

### Training crashed mid-run

Check the log for a Python traceback:
```bash
kubectl apply -f nrp/pvc_stage_pod.yaml
kubectl exec -n ssu-intelligent-systems pvc-stage -- \
  tail -200 /workspace/logs/cube_train.log
kubectl delete pod pvc-stage -n ssu-intelligent-systems
```
The last saved checkpoint before the crash is `latest.ckpt`. Contact Prof. Shrestha to set up resuming.

---

## Quick Reference

```bash
# Submit
kubectl apply -f nrp/umi_train_<your_name>_v1.yaml

# Watch status
kubectl get pods -n ssu-intelligent-systems -w

# Get pod name
POD=$(kubectl get pods -n ssu-intelligent-systems \
      -l job-name=umi-train-<your_name>-v1 \
      -o jsonpath='{.items[0].metadata.name}')

# Stream log
kubectl exec -n ssu-intelligent-systems $POD -- tail -f /workspace/logs/cube_train.log

# List checkpoints
kubectl exec -n ssu-intelligent-systems $POD -- \
  ls /workspace/checkpoints/<your_name>_001/checkpoints/

# Delete finished job before resubmitting
kubectl delete job umi-train-<your_name>-v1 -n ssu-intelligent-systems
```
