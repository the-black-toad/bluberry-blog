---
layout: post
title: "Building a Blueberry Picker, Part 2: Dataset Capture and Labeling"
date: 2026-05-19
---

# Building a Blueberry Picker, Part 2: Dataset Capture and Labeling

*May 19, 2026*

Last time I got three webcams calibrated and fused into a single point cloud. That's the geometry side of the problem solved. Now it's time to teach the robot what a blueberry actually looks like.

## The Seasonal Timing Problem

It's late May and blueberries aren't ripe yet — we're still a few weeks out. The temptation was to just wait, but that would mean sitting idle during the most productive part of the build window. So the plan became: capture unripe berries now to bootstrap the dataset, then shoot ripe berries when the season hits and fold them in.

This is actually fine for training. YOLO doesn't care what order the classes arrive in, and having unripe examples early means the labeling infrastructure (Roboflow account, class definitions, augmentation pipeline) is all set up and tested before the ripe-season crunch.

## 116 Photos, 9,818 Berries

I went out and shot 116 images of unripe blueberry bushes with a DSLR. The goal was variety: different lighting conditions, angles, distances, and levels of occlusion. Clusters half-hidden behind leaves, berries slightly out of focus in the foreground, tight shots, wide context shots — the model will see all of these at inference time so it should see them in training too.

One tradeoff worth noting: the inference pipeline runs on C930e/C920 webcam frames at 1024×576, not DSLR images. There's a domain gap there — sensor noise, color response, and sharpness all differ. The plan is to supplement with webcam-captured images when ripe season arrives, which will be the more important class anyway.

For labeling I used Roboflow's Smart Select tool, which traces berry contours automatically. After a quick pass to catch anything it missed (especially small or partially occluded berries in the background), I ended up with **9,818 labeled instances** across the 116 images — roughly 85 berries per image, which tracks for dense cluster shots.

The single class right now is `blueberry`. When ripe and overripe examples come in, I'll add `ripe` and `unripe` as distinct classes and relabel or split accordingly.

## Dataset Version + Augmentations

Roboflow makes it easy to generate a versioned, augmented dataset from your labeled images. The augmentations I applied:

- **Horizontal and vertical flip** — blueberries on a bush look the same from any orientation
- **Rotation ±15°** — the rig won't always be perfectly level
- **Brightness ±25%** — outdoor lighting swings hard
- **Blur up to 2px** — the dataset already has natural blur from depth of field; this adds a bit more for robustness

The dataset was exported in **YOLOv8 format**, ready for fine-tuning.

## HSV Baseline Detector

Before training any neural network, I wrote a classical baseline detector (`detection/hsv_baseline.py`). The idea: color-threshold the image in HSV space and filter contours by area and circularity. Blueberries are roughly round and have fairly distinctive colors at each ripeness stage — green for unripe, blue-purple for ripe.

The script has two modes:

```bash
# Interactive tuning — trackbars to dial in HSV ranges on a single image
python detection/hsv_baseline.py tune images/IMG_3393.JPG

# Batch eval — annotated images saved to images/hsv_detections/
python detection/hsv_baseline.py eval images/

# With labels — computes precision / recall / F1
python detection/hsv_baseline.py eval images/ --labels images/labels/
```

Once the Roboflow labels are exported locally, I'll run the eval mode to get a precision/recall number. That becomes the floor the YOLO model has to beat — if fine-tuned YOLO can't outperform simple color thresholding, something is wrong with the training pipeline.

## What's Next

The dataset is ready. Next session: fine-tune YOLOv8n on the GPU desktop and see what the model gets on the validation set. If it looks good, the session after that hooks detection into the live webcam streams and starts the 2D→3D localization work.

Phase 1 is now 16/22 tasks complete. Getting close.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
