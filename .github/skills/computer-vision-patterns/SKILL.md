---
name: computer-vision-patterns
description: Computer vision patterns for object detection (YOLO), image processing (OpenCV), OCR, video analysis, model export, and edge deployment.
---

# Computer Vision Development Patterns

Patterns for building robust computer vision pipelines — from data preparation through deployment.

## When to Activate

- Building object detection pipelines (YOLO, Faster R-CNN)
- Processing images with OpenCV or PIL/Pillow
- Implementing OCR or text recognition
- Training or fine-tuning vision models
- Deploying vision models (ONNX, TensorRT, edge devices)
- Working with video streams or real-time detection

## Core Principles

### 1. Preprocessing Consistency

Training and inference preprocessing MUST be identical. Mismatches are the #1 cause of accuracy drops.

```python
# Define once, use everywhere
class ImagePreprocessor:
    def __init__(self, target_size: tuple[int, int] = (640, 640)):
        self.target_size = target_size
        self.mean = np.array([0.485, 0.456, 0.406])
        self.std = np.array([0.229, 0.224, 0.225])

    def __call__(self, image: np.ndarray) -> np.ndarray:
        """Preprocess image for model input."""
        # OpenCV loads BGR — convert to RGB
        if len(image.shape) == 3 and image.shape[2] == 3:
            image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)

        # Letterbox resize (preserve aspect ratio)
        image = self.letterbox(image, self.target_size)

        # Normalize
        image = image.astype(np.float32) / 255.0
        image = (image - self.mean) / self.std

        # HWC → CHW for PyTorch
        image = np.transpose(image, (2, 0, 1))
        return image
```

### 2. BGR vs RGB Awareness

OpenCV uses BGR. PIL uses RGB. Most models expect RGB. Always be explicit.

```python
# OpenCV → RGB (for model input)
img_bgr = cv2.imread("image.jpg")
img_rgb = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2RGB)

# PIL → OpenCV
from PIL import Image
pil_img = Image.open("image.jpg")
cv_img = cv2.cvtColor(np.array(pil_img), cv2.COLOR_RGB2BGR)

# OpenCV → PIL
pil_img = Image.fromarray(cv2.cvtColor(cv_img, cv2.COLOR_BGR2RGB))
```

### 3. Coordinate System Consistency

Different formats for bounding boxes — always document and convert explicitly.

```python
def xyxy_to_xywh(box: tuple) -> tuple:
    """Convert (x1, y1, x2, y2) to (x_center, y_center, width, height)."""
    x1, y1, x2, y2 = box
    return ((x1 + x2) / 2, (y1 + y2) / 2, x2 - x1, y2 - y1)

def xywh_to_xyxy(box: tuple) -> tuple:
    """Convert (x_center, y_center, width, height) to (x1, y1, x2, y2)."""
    cx, cy, w, h = box
    return (cx - w / 2, cy - h / 2, cx + w / 2, cy + h / 2)

# YOLO format: normalized (x_center, y_center, w, h) relative to image size
def yolo_to_xyxy(box: tuple, img_w: int, img_h: int) -> tuple:
    cx, cy, w, h = box
    x1 = (cx - w / 2) * img_w
    y1 = (cy - h / 2) * img_h
    x2 = (cx + w / 2) * img_w
    y2 = (cy + h / 2) * img_h
    return (x1, y1, x2, y2)
```

## Object Detection with YOLO

### Training Configuration

```yaml
# dataset.yaml
path: /data/license_plates
train: images/train
val: images/val
test: images/test

names:
  0: plate

# Training
# model = YOLO("yolov8n.pt")
# model.train(data="dataset.yaml", epochs=100, imgsz=640, batch=16)
```

### Detection Pipeline

```python
from ultralytics import YOLO

class PlateDetector:
    def __init__(self, model_path: str, conf: float = 0.25, iou: float = 0.45):
        self.model = YOLO(model_path)
        self.conf = conf
        self.iou = iou

    def detect(self, image: np.ndarray) -> list[dict]:
        """Detect objects and return structured results."""
        results = self.model.predict(
            source=image,
            conf=self.conf,
            iou=self.iou,
            verbose=False,
        )

        detections = []
        for result in results:
            for box in result.boxes:
                detections.append({
                    "bbox": box.xyxy[0].cpu().numpy().tolist(),
                    "confidence": float(box.conf[0]),
                    "class_id": int(box.cls[0]),
                    "class_name": result.names[int(box.cls[0])],
                })
        return detections
```

### Video Processing

```python
def process_video(
    video_path: str,
    detector,
    frame_skip: int = 1,
    output_path: str | None = None,
) -> list[dict]:
    """Process video with detection, optionally writing output."""
    cap = cv2.VideoCapture(video_path)
    if not cap.isOpened():
        raise ValueError(f"Cannot open video: {video_path}")

    fps = cap.get(cv2.CAP_PROP_FPS)
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))

    writer = None
    if output_path:
        fourcc = cv2.VideoWriter_fourcc(*"mp4v")
        writer = cv2.VideoWriter(output_path, fourcc, fps, (width, height))

    all_detections = []
    frame_idx = 0

    try:
        while True:
            ret, frame = cap.read()
            if not ret:
                break

            if frame_idx % frame_skip == 0:
                detections = detector.detect(frame)
                all_detections.append({"frame": frame_idx, "detections": detections})

                if writer:
                    annotated = draw_detections(frame, detections)
                    writer.write(annotated)
            elif writer:
                writer.write(frame)

            frame_idx += 1
    finally:
        cap.release()
        if writer:
            writer.release()

    return all_detections
```

## OCR Patterns

### License Plate OCR Pipeline

```python
class LicensePlateOCR:
    def __init__(self, detector_path: str, ocr_model_path: str):
        self.detector = PlateDetector(detector_path)
        self.ocr_model = YOLO(ocr_model_path)

    def read_plate(self, image: np.ndarray) -> list[dict]:
        """Detect plates and read text."""
        plates = self.detector.detect(image)
        results = []

        for plate in plates:
            x1, y1, x2, y2 = [int(c) for c in plate["bbox"]]
            crop = image[y1:y2, x1:x2]

            # Preprocess for OCR
            crop = self.preprocess_for_ocr(crop)

            # Run OCR
            text = self.recognize_text(crop)

            results.append({
                "bbox": plate["bbox"],
                "text": text,
                "plate_confidence": plate["confidence"],
            })
        return results

    @staticmethod
    def preprocess_for_ocr(image: np.ndarray) -> np.ndarray:
        """Preprocess plate crop for OCR."""
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        # Adaptive thresholding for varying lighting
        binary = cv2.adaptiveThreshold(
            gray, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv2.THRESH_BINARY, 11, 2
        )
        # Denoise
        denoised = cv2.fastNlMeansDenoising(binary, h=10)
        return denoised
```

## Image Processing Utilities

### Letterbox Resize

```python
def letterbox(
    image: np.ndarray,
    new_shape: tuple[int, int] = (640, 640),
    color: tuple[int, int, int] = (114, 114, 114),
) -> np.ndarray:
    """Resize image with letterboxing (preserve aspect ratio)."""
    h, w = image.shape[:2]
    r = min(new_shape[0] / h, new_shape[1] / w)
    new_unpad = (int(round(w * r)), int(round(h * r)))

    if (w, h) != new_unpad:
        image = cv2.resize(image, new_unpad, interpolation=cv2.INTER_LINEAR)

    dw = new_shape[1] - new_unpad[0]
    dh = new_shape[0] - new_unpad[1]
    top, bottom = dh // 2, dh - dh // 2
    left, right = dw // 2, dw - dw // 2

    image = cv2.copyMakeBorder(
        image, top, bottom, left, right,
        cv2.BORDER_CONSTANT, value=color
    )
    return image
```

### Drawing Utilities

```python
def draw_detections(
    image: np.ndarray,
    detections: list[dict],
    color: tuple[int, int, int] = (0, 255, 0),
    thickness: int = 2,
) -> np.ndarray:
    """Draw bounding boxes and labels on image."""
    annotated = image.copy()
    for det in detections:
        x1, y1, x2, y2 = [int(c) for c in det["bbox"]]
        label = f"{det.get('class_name', '')} {det['confidence']:.2f}"

        cv2.rectangle(annotated, (x1, y1), (x2, y2), color, thickness)

        # Label background
        (tw, th), _ = cv2.getTextSize(label, cv2.FONT_HERSHEY_SIMPLEX, 0.6, 1)
        cv2.rectangle(annotated, (x1, y1 - th - 8), (x1 + tw, y1), color, -1)
        cv2.putText(annotated, label, (x1, y1 - 4),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 1)
    return annotated
```

## Model Export & Deployment

```python
# YOLO export to multiple formats
model = YOLO("best.pt")

# ONNX (cross-platform)
model.export(format="onnx", opset=17, simplify=True, dynamic=True)

# TensorRT (NVIDIA GPU inference)
model.export(format="engine", half=True, device=0)

# CoreML (Apple devices)
model.export(format="coreml", nms=True)

# TFLite (mobile/edge)
model.export(format="tflite", int8=True)
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| BGR/RGB mismatch | Always convert explicitly and document |
| Different preprocessing for train/inference | Share single preprocessing class |
| Hardcoded image size | Use config, match train and inference |
| Processing every video frame | Skip frames based on FPS and use case |
| No NMS post-processing | Always apply NMS to raw detections |
| Ignoring aspect ratio on resize | Use letterbox resize |
| Single confidence threshold | Expose as configurable parameter |
| No GPU memory management | Use `torch.cuda.empty_cache()` between batches |
