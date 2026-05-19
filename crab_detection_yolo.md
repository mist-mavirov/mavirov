# 🦀 Crab Detection — YOLO

YOLO object detection project for classifying three crab species: **Green**, **Jonah**, and **Rock**. Dataset exported from Roboflow in YOLO format.

## Code

```python
from ultralytics import YOLO

# Load a YOLO model
model = YOLO("yolo11n.pt")

# Train on the crab dataset
model.train(
    data="data.yaml",
    epochs=100,
    imgsz=640,
    batch=8,
    name="crab_detector"
)

# Validate
metrics = model.val()

# Predict on a new image
results = model("path/to/crab_photo.jpg")

for r in results:
    r.show()
    r.save("output.jpg")
    for box in r.boxes:
        cls_name = model.names[int(box.cls)]
        conf = float(box.conf)
        print(f"{cls_name}: {conf:.2f}")
```

## `data.yaml`

```yaml
train: ../train/images
val: ../valid/images
test: ../test/images

nc: 3
names: ['Green', 'Jonah', 'Rock']
```
