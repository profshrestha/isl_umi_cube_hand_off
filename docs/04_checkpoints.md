# Step 4 — Downloading Checkpoints

When training completes (or at any point during training), you can download checkpoints from the NRP PVC to your local machine.

---

## What's in the Checkpoint Directory

```
/workspace/checkpoints/<your_name>_001/
├── checkpoints/
│   ├── epoch=0000-train_loss=0.030.ckpt
│   ├── epoch=0010-train_loss=0.014.ckpt
│   ├── epoch=0020-train_loss=0.013.ckpt
│   ├── ...
│   └── latest.ckpt                        # symlink to the most recent checkpoint
└── logs.json.txt                           # per-epoch loss history
```

Checkpoints are saved every 10 epochs. `latest.ckpt` always points to the most recently saved checkpoint, whether or not training has finished.

---

## Step 4.1 — List Available Checkpoints

If the training pod is still running:
```bash
POD=$(kubectl get pods -n ssu-intelligent-systems \
      -l job-name=umi-train-<your_name>-v1 \
      -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n ssu-intelligent-systems $POD -- \
  ls -lh /workspace/checkpoints/<your_name>_001/checkpoints/
```

If the pod has finished (status `Completed` or deleted), use the staging pod:
```bash
kubectl apply -f nrp/pvc_stage_pod.yaml
kubectl exec -n ssu-intelligent-systems pvc-stage -- \
  ls -lh /workspace/checkpoints/<your_name>_001/checkpoints/
kubectl delete pod pvc-stage -n ssu-intelligent-systems
```

---

## Step 4.2 — Download the Latest Checkpoint

### If the training pod is still Running

```bash
POD=$(kubectl get pods -n ssu-intelligent-systems \
      -l job-name=umi-train-<your_name>-v1 \
      -o jsonpath='{.items[0].metadata.name}')

mkdir -p ~/checkpoints/<your_name>

kubectl cp \
  ssu-intelligent-systems/$POD:/workspace/checkpoints/<your_name>_001/checkpoints/latest.ckpt \
  ~/checkpoints/<your_name>/latest.ckpt
```

### If the training pod has finished

Use the staging pod to access the PVC:
```bash
kubectl apply -f nrp/pvc_stage_pod.yaml
kubectl wait --for=condition=Ready pod/pvc-stage -n ssu-intelligent-systems --timeout=120s

mkdir -p ~/checkpoints/<your_name>

kubectl cp \
  ssu-intelligent-systems/pvc-stage:/workspace/checkpoints/<your_name>_001/checkpoints/latest.ckpt \
  ~/checkpoints/<your_name>/latest.ckpt

kubectl delete pod pvc-stage -n ssu-intelligent-systems
```

---

## Step 4.3 — Download a Specific Epoch Checkpoint

Replace `latest.ckpt` with the specific filename:

```bash
kubectl cp \
  ssu-intelligent-systems/pvc-stage:/workspace/checkpoints/<your_name>_001/checkpoints/epoch=0100-train_loss=0.011.ckpt \
  ~/checkpoints/<your_name>/epoch_100.ckpt
```

---

## Step 4.4 — Download the Loss History

The `logs.json.txt` file contains per-epoch train and validation loss — useful for plotting a learning curve:

```bash
kubectl cp \
  ssu-intelligent-systems/pvc-stage:/workspace/checkpoints/<your_name>_001/logs.json.txt \
  ~/checkpoints/<your_name>/logs.json.txt
```

Quick plot with Python:
```python
import json, matplotlib.pyplot as plt

losses = [json.loads(l) for l in open('logs.json.txt') if l.strip()]
epochs = [x['epoch'] for x in losses if 'train_loss' in x]
train  = [x['train_loss'] for x in losses if 'train_loss' in x]

plt.plot(epochs, train)
plt.xlabel('Epoch'); plt.ylabel('Train Loss')
plt.title('Diffusion Policy Training Curve')
plt.savefig('loss_curve.png')
```

---

## What to Do with the Checkpoint

The `.ckpt` file is a PyTorch Lightning checkpoint containing the full model weights. Deployment (running inference on the real RX150 arms) requires additional setup — contact Prof. Shrestha for the deployment pipeline.

To verify the checkpoint loaded correctly on the lab workstation:

```bash
conda run -n umi python - <<'EOF'
import torch
ckpt = torch.load('~/checkpoints/<your_name>/latest.ckpt', map_location='cpu')
print("Keys:", list(ckpt.keys()))
print("Epoch:", ckpt.get('epoch'))
EOF
```
