---
layout: post
title: "Building a Blueberry Picker, Part 4: 3D Berry Localization"
date: 2026-05-26
categories: [Blueberry Picker, Depth Estimation]
tags: [open3d, point-cloud, stereo, yolo, 3d-localization]
---

# Building a Blueberry Picker, Part 4: 3D Berry Localization

*May 26, 2026*

The previous session had a model that could draw bounding boxes around blueberries in a 2D image. This session: figure out where those berries actually are in space.

## The Problem

YOLO gives you a pixel coordinate — the centroid of a bounding box in a rectified camera image. The stereo pipeline gives you a disparity map where each pixel encodes how far left or right the same point appears between the two cameras. Combine them and you get a 3D position. In principle this is straightforward; in practice there are a few things to get right.

## From Pixel to Point

The approach in `detection/detect_3d.py`:

1. **Rectify** both cameras in a stereo pair using the precomputed maps from `stereoRectify`.
2. **Run YOLO** on the rectified left image.
3. For each detection, **sample a 7×7 pixel patch** around the bounding-box centroid in the disparity map and take the median of valid (positive) values. This makes the depth estimate robust to the holes SGBM leaves around edges and low-texture regions.
4. **Project** the centroid through the Q matrix: `p = Q @ [cx, cy, d, 1]`, then divide by the homogeneous coordinate. This gives (x, y, z) in millimeters in the rectified-left frame.
5. **Undo the rectification rotation**: `p_orig = R1.T @ p_rect`. The Q matrix puts points in the rectified frame, but the extrinsics live in the original camera frames.
6. **Transform to CAM_A's frame** if the left camera is not CAM_A (for the narrow_left pair where CAM_C is the left camera, apply the CAM_C→CAM_A extrinsic R, T).

The script runs two stereo pairs simultaneously using a thread pool — one for narrow_right (CAM_A left, CAM_B right) and one for narrow_left (CAM_C left, CAM_A right). SGBM releases the GIL during computation so both pairs actually run in parallel.

## Clustering Across Views

A berry visible to both pairs produces two detections that should merge into one. The clustering is a simple greedy nearest-neighbor pass: for each new detection, if there is already a cluster whose 3D centroid is within 30mm, update that cluster's position as a running mean and keep the max confidence. Otherwise start a new cluster. The 30mm threshold is larger than typical per-frame noise but small enough that adjacent berries a centimeter apart don't merge.

The result is a list of unique 3D berry positions, sorted by depth.

## Live Display

The two annotated views are shown side-by-side with the berry list overlaid in the top-left corner:

```
fps=4.2  berries=1
  #1  (-26, +16, 351) mm  n=3
```

`n` is the number of detections that merged into that cluster — seeing the same berry from both pairs before clustering gives n=2 or 3, which is a useful sanity check that the cross-view transform is working.

## Validation in Open3D

`processing/visualize_berries.py` runs a heavier validation pass. Press SPACE to freeze a frame set; the script then:

1. Builds the full **fused point cloud** from all three stereo pairs (the same pipeline as `fuse_pointclouds.py`).
2. Runs the **YOLO 3D detection** on the two left-camera pairs.
3. Opens an **Open3D window** with the colored cloud, a coordinate frame at the CAM_A origin, and a **25mm red sphere** at each detected berry centroid.

The window is interactive — you can orbit and zoom — and a screenshot is automatically saved before you interact with it.

The first validation result with a checkerboard placed behind a cluster of berries:

```
#1  xyz=(-26.5, +15.9, 351.0) mm  conf=0.61  views=2
#2  xyz=(+144.6, +104.6, 535.6) mm  conf=0.49  views=1
```

The depth for berry #1 is stable to within a couple of millimeters across independent runs. The red spheres land on the checkerboard surface in the point cloud, at the correct spatial positions relative to the background plane.

The checkerboard background turned out to be useful during testing: SGBM's block-matching needs texture to compute disparity, and the board's high-contrast pattern means dense, clean depth readings right behind the berries — fewer holes in the disparity map directly around the detection centroids.

## What the Numbers Mean

At 350mm depth with an ~100mm baseline (narrow pair), each disparity unit corresponds to roughly 1mm of depth error near the centroid. The 7×7 median patch averages out pixel-level noise. In practice the x/y coordinates shift by ±5–10mm between frames and the z coordinate is stable to about ±2mm — more than accurate enough for the arm to locate a 12mm-radius berry.

## What's Next

Phase 1 is 21/22 tasks complete. The last task is the retrospective: what worked, what didn't, how the time estimates held up, and the go/no-go decision for Phase 2 (the arm).

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
