---
layout: post
title: "Building a Blueberry Picker, Part 3: Training YOLOv8 and Running Live Detection"
date: 2026-05-20
categories: [Blueberry Picker, Machine Learning]
tags: [yolo, yolov8, object-detection, training, inference]
---

Last time I had a labeled dataset but no model. This session: train YOLOv8n, get it running on the live camera rig, and see if it actually detects blueberries in the real world.

## Dataset Splits and a Common Roboflow Gotcha

The first attempt at training failed before it started. When I exported the dataset from Roboflow, all 348 images ended up in the `train/` split — Roboflow requires you to explicitly assign splits before generating a dataset version, otherwise everything lands in training. Easy fix: configure an 70/20/10 split in the Roboflow UI and regenerate. The resulting v2 export had 243 train / 23 val / 12 test images.

There's also a path issue worth noting. Roboflow exports `data.yaml` with paths like `../train/images`, relative to a parent directory that doesn't exist in the typical clone-and-train setup. The fix is simple — change them to `train/images` — but it'll silently break training if you miss it.

## Training on a GPU Desktop

YOLOv8n trains in minutes on a GPU; it would take hours on CPU. I transferred the dataset to a desktop with a CUDA-capable GPU, set up a fresh venv, and installed Ultralytics. One thing to check before using a `requirements.txt` from a CPU development machine: it may contain `torch==X.Y.Z+cpu`. Installing that on the training machine will silently run on CPU. Verify with:

```python
import torch
print(torch.cuda.is_available())
```

Training was a single command:

```bash
yolo train model=yolov8n.pt data=data.yaml epochs=100 imgsz=640 batch=16
```

## Results

| Metric | Value |
|--------|-------|
| mAP50 | 0.678 |
| Precision | 0.812 |
| Recall | 0.597 |
| mAP50-95 | 0.438 |

Precision at 81% is solid — when the model calls something a blueberry, it's usually right. Recall at 60% is the weak spot: it's missing roughly 4 in 10 berries. That's expected and acceptable at this stage. The entire dataset is unripe blueberries shot with a DSLR. Ripe berries (the class that actually matters for picking) aren't in the training data yet, and webcam-captured images are underrepresented. When harvest season arrives and I add a ripe class with webcam images, recall should improve significantly. For now, 68% mAP50 is a useful baseline.

The trained weights (`best.pt`, ~6MB) were committed directly to the repo since they're small enough not to warrant git-lfs.

## Live Inference on the Camera Rig

`detection/yolo_detect.py` runs the model on a live camera feed:

```bash
python detection/yolo_detect.py CAM_A
```

It uses the same `cameras.py` stable USB path lookup that the stereo pipeline uses, so it survives camera replug without needing to update device indices. Press `s` to save an annotated frame, `q` to quit.

The first test was encouraging: I picked a few unripe berries from the yard and held them up to CAM_A. The model picked them up immediately with reasonable confidence scores. The detections tracked well as I moved the berries around. Not a rigorous benchmark, but good confirmation that the model generalized beyond the training images.

## What the HSV Baseline Tells Us

The HSV baseline detector (`detection/hsv_baseline.py`) was built before training as a performance floor. It runs three color ranges in HSV space (green unripe, pink/reddish unripe, blue-purple ripe), then filters by contour circularity and area. It's a useful sanity check: if fine-tuned YOLO can't beat blob detection, something is wrong with the training pipeline. Full precision/recall comparison against the labeled set is still pending, but initial runs suggest the YOLO model is clearly ahead on crowded cluster shots where the HSV approach over-segments.

## What's Next

The model detects blueberries in 2D. The stereo pipeline computes depth for every pixel. The next step joins them: for each detected bounding box, look up the depth at the centroid pixel, project through the Q matrix, and get an (x, y, z) position in world coordinates. After clustering detections across all three views, the output is a list of 3D berry positions — the input the picking arm will eventually consume.

Phase 1 is now 19/22 tasks complete.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
