---
name: ai-ml-patterns
description: Machine learning lifecycle patterns, model training, inference optimization, data pipelines, experiment tracking, and MLOps best practices for PyTorch, Ultralytics, scikit-learn, and TensorFlow.
---

# AI/ML Development Patterns

End-to-end machine learning patterns for building robust, reproducible, and production-ready ML systems.

## When to Activate

- Designing ML training pipelines or model architectures
- Implementing data preprocessing and augmentation
- Optimizing model inference (latency, memory, throughput)
- Setting up experiment tracking and model versioning
- Deploying models to production (serving, monitoring, A/B testing)
- Evaluating model performance and debugging degradation

## Core Principles

### 1. Reproducibility First

Every experiment must be reproducible. Fix seeds, log hyperparameters, version data and code.

```python
import torch
import random
import numpy as np

def set_seed(seed: int = 42) -> None:
    """Set all random seeds for reproducibility."""
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False
```

### 2. Data-Centric Over Model-Centric

Improving data quality yields more gains than architecture tweaks.

```python
# Good: Systematic data quality checks
def validate_dataset(dataset_path: str) -> dict:
    """Run data quality checks before training."""
    issues = {
        "missing_labels": [],
        "corrupt_images": [],
        "class_imbalance": {},
        "duplicate_images": [],
    }
    # Check each sample...
    return issues
```

### 3. Fail Fast With Clear Diagnostics

Validate all assumptions before expensive training runs.

```python
def pre_training_checks(model, dataloader, loss_fn):
    """Run sanity checks before full training."""
    # 1. Verify model can overfit a single batch
    batch = next(iter(dataloader))
    for _ in range(100):
        loss = train_step(model, batch, loss_fn)
    assert loss < 0.1, f"Cannot overfit single batch: loss={loss:.4f}"

    # 2. Verify loss is reasonable at init
    model_fresh = create_model()
    init_loss = evaluate(model_fresh, dataloader)
    expected = -math.log(1.0 / num_classes)  # Random baseline
    assert abs(init_loss - expected) < 0.5, f"Init loss {init_loss:.4f} != expected {expected:.4f}"
```

## Training Patterns

### Training Loop Template

```python
def train_epoch(
    model: nn.Module,
    dataloader: DataLoader,
    optimizer: torch.optim.Optimizer,
    loss_fn: nn.Module,
    device: torch.device,
    scaler: torch.GradScaler | None = None,
) -> dict[str, float]:
    model.train()
    total_loss = 0.0
    num_batches = 0

    for batch in dataloader:
        inputs = batch["image"].to(device, non_blocking=True)
        targets = batch["label"].to(device, non_blocking=True)

        optimizer.zero_grad(set_to_none=True)  # More efficient than zero_grad()

        if scaler:  # Mixed precision
            with torch.autocast(device_type="cuda"):
                outputs = model(inputs)
                loss = loss_fn(outputs, targets)
            scaler.scale(loss).backward()
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            scaler.step(optimizer)
            scaler.update()
        else:
            outputs = model(inputs)
            loss = loss_fn(outputs, targets)
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()

        total_loss += loss.item()
        num_batches += 1

    return {"train_loss": total_loss / num_batches}
```

### Learning Rate Scheduling

```python
# Warmup + cosine annealing
from torch.optim.lr_scheduler import CosineAnnealingLR, LinearLR, SequentialLR

warmup = LinearLR(optimizer, start_factor=0.01, total_iters=warmup_epochs)
cosine = CosineAnnealingLR(optimizer, T_max=total_epochs - warmup_epochs, eta_min=1e-6)
scheduler = SequentialLR(optimizer, schedulers=[warmup, cosine], milestones=[warmup_epochs])
```

### Early Stopping

```python
class EarlyStopping:
    def __init__(self, patience: int = 10, min_delta: float = 0.001):
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_score: float | None = None

    def should_stop(self, val_metric: float) -> bool:
        if self.best_score is None or val_metric > self.best_score + self.min_delta:
            self.best_score = val_metric
            self.counter = 0
            return False
        self.counter += 1
        return self.counter >= self.patience
```

## Data Pipeline Patterns

### Efficient DataLoader Configuration

```python
dataloader = DataLoader(
    dataset,
    batch_size=batch_size,
    shuffle=True,              # Only for training
    num_workers=min(8, os.cpu_count()),
    pin_memory=True,           # Faster CPU→GPU transfer
    persistent_workers=True,   # Reuse worker processes
    prefetch_factor=2,         # Prefetch 2 batches per worker
    drop_last=True,            # Consistent batch sizes
)
```

### Data Augmentation Pipeline

```python
import albumentations as A
from albumentations.pytorch import ToTensorV2

train_transform = A.Compose([
    A.RandomResizedCrop(640, 640, scale=(0.5, 1.0)),
    A.HorizontalFlip(p=0.5),
    A.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2, hue=0.1),
    A.GaussNoise(var_limit=(10, 50), p=0.3),
    A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ToTensorV2(),
], bbox_params=A.BboxParams(format="yolo", label_fields=["class_labels"]))

val_transform = A.Compose([
    A.Resize(640, 640),
    A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ToTensorV2(),
])
```

## Inference Optimization

### Batch Inference

```python
@torch.inference_mode()
def batch_predict(model: nn.Module, images: list[np.ndarray], batch_size: int = 32) -> list:
    model.eval()
    results = []
    for i in range(0, len(images), batch_size):
        batch = torch.stack([preprocess(img) for img in images[i:i+batch_size]])
        batch = batch.to(device)
        outputs = model(batch)
        results.extend(postprocess(outputs))
    return results
```

### Model Export

```python
# ONNX export for deployment
torch.onnx.export(
    model, dummy_input,
    "model.onnx",
    opset_version=17,
    input_names=["input"],
    output_names=["output"],
    dynamic_axes={"input": {0: "batch"}, "output": {0: "batch"}},
)
```

## Experiment Tracking

```python
# Minimal experiment logging
import json
from pathlib import Path
from datetime import datetime

def log_experiment(config: dict, metrics: dict, output_dir: str) -> None:
    experiment = {
        "timestamp": datetime.now().isoformat(),
        "config": config,
        "metrics": metrics,
        "git_hash": get_git_hash(),
    }
    path = Path(output_dir) / "experiment_log.jsonl"
    with open(path, "a") as f:
        f.write(json.dumps(experiment) + "\n")
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Training without validation split | Always hold out ≥10% for validation |
| No learning rate schedule | Use warmup + cosine decay |
| Evaluating on augmented data | Augment only training data |
| Ignoring class imbalance | Use weighted loss or oversampling |
| Single metric evaluation | Track precision, recall, F1, and task-specific metrics |
| No checkpointing | Save best and last checkpoints |
| Hardcoded hyperparameters | Use config files or CLI args |
| No seed setting | Fix all random seeds |
