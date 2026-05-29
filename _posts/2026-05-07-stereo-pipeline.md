---
layout: post
title: "Pixels Become Points"
date: 2026-05-07 12:38:00 -0400
categories: [Computer Vision, Depth Estimation]
tags: [stereo, disparity, point-cloud, sgbm, open3d]
---

# Pixels Become Points

*Or: how I built a working stereo depth pipeline, then spent half a day learning that side views of point clouds are a lie.*

This is the third post in a series documenting a multi-camera point cloud project. The [first post](setup_and_calibration.md) covered environment setup and per-camera intrinsic calibration. The [second](extrinsic_calibration.md) covered extrinsic calibration — the spatial relationships between three cameras. This one covers the payoff: turning all that calibration into actual depth measurements and 3D points.

The result is a working pipeline that produces a recognizable 3D representation of any scene the cameras look at, and a meaningful lesson about what stereo vision can and cannot do.

## The pipeline at thirty thousand feet

Four stages, each building on the last:

**Rectification** — given two cameras with known intrinsics and extrinsics, compute pixel-level remapping tables that warp both images so corresponding points lie on the same horizontal scanline. Without this, finding matching points between the two cameras is a 2D search problem; with it, you slide along one row.

**Disparity** — for each pixel in the rectified left image, find the corresponding pixel in the rectified right image and compute their horizontal offset. Closer objects = larger offset. OpenCV's StereoSGBM (Semi-Global Block Matching) is the standard algorithm.

**Depth** — convert disparity to actual distance in millimeters. The math: `depth = baseline × focal_length / disparity`. The Q matrix from rectification handles this through a single matrix multiplication.

**Fusion** — repeat the above for each of the three camera pairs (close-left, close-right, wide). Transform all the resulting points into a single shared coordinate frame. Combine.

Each stage is a working OpenCV one-liner. The interesting work is in the parts surrounding the one-liners.

## Stage 1: Rectification

Conceptually clean, mechanically straightforward, but worth slowing down on.

Each pair of cameras has slightly different optical orientations. The pair was calibrated to find the rotation and translation between them, but the cameras are still pointing in different directions in the world. Rectification computes a pair of virtual rotations — `R1` for the left camera, `R2` for the right — that, when applied, make both cameras' image planes parallel and their image rows aligned.

After rectification:
- Both rectified cameras share a common image plane
- Both have the same focal length (the average of the two original focal lengths, roughly)
- Corresponding points lie on the same y-coordinate in both images

This means the disparity search becomes 1D. For each pixel in the left rectified image, the corresponding pixel in the right is somewhere along the same row. Instead of searching a 2D area, you slide along a line.

The rectification machinery in OpenCV produces precomputed lookup tables (`map1x, map1y, map2x, map2y`) that warp original camera images to rectified images via a single `cv2.remap` call. Compute the maps once per pair, save them, use them for every frame.

A useful sanity check: in the rectified images, if you draw horizontal scan lines and look at any feature in the scene, it should land on the same line in both images even if it's shifted left/right. This works as a manual verification — if the scan lines disagree, your calibration has issues. Mine agreed cleanly.

## Stage 2: Disparity

This is where the math meets reality.

StereoSGBM has a roomful of parameters. Most people copy default values from a tutorial and don't think about them. They are, in fact, all important and can ruin your results subtly:

**numDisparities**: maximum disparity searched. Higher = can detect closer objects, slower computation. Must be divisible by 16.

**blockSize**: matching window size. Larger = smoother but loses detail. 5-9 is typical.

**P1, P2**: penalties for disparity changes between neighboring pixels. Should scale with `blockSize²`.

**uniquenessRatio**: how much better the best match must be than the second-best. Higher = stricter.

**speckleWindowSize, speckleRange**: post-process cleanup of small noisy regions.

I burned an hour tuning these on a live trackbar UI — pointing the rig at a textured scene (chairs and a cabinet), watching the disparity map update as I dragged sliders. Useful exercise, both for getting decent values and for developing intuition about what each parameter does.

The biggest gotcha is **numDisparities**. With my wide pair (8" baseline) and `numDisparities=128`, the algorithm could only see down to about 1m. Anything closer had disparity exceeding 128 pixels and got marked as no-match. The classic symptom: disparity flickered between "valid" and "no depth" at exactly the threshold distance. Bumping to 256 fixed that for the wide pair; close-range work is still better handled by the narrow pairs (4" baseline, where the same `numDisparities` covers down to ~30cm).

The output of disparity computation is a 2D array: same shape as the input image, with each pixel containing the horizontal offset to the matching point in the other image. Or -1 if no match was found.

## Stage 3: Depth

Trivial, given the previous stages.

`cv2.reprojectImageTo3D(disparity_map, Q)` produces an `(H, W, 3)` array where each pixel is a 3D point in the rectified left camera's frame, in millimeters (because we used millimeters for the calibration board square size).

The Q matrix encodes the camera's geometry. Skipping the math, you end up with `depth = baseline × fx / disparity`, but expressed as a matrix multiplication that also gives you proper X and Y coordinates.

This is the moment where pixels become points. You can now ask "what 3D position does this pixel correspond to" and get an answer in millimeters.

## Verifying with a tape measure

There's a special joy in the moment when the calibration math first interacts with physical reality. I added a live readout to the script: a yellow crosshair at the image center, and a "depth: XXXmm" overlay. Then:

- Held a textured object at exactly 1m. Readout: 980mm.
- 2m. Readout: 1.95m.
- 3m. Readout: 2.85m.

Within a few percent at every distance. The whole calibration chain — intrinsics, extrinsics, rectification, disparity, Q matrix — is producing real-world distances.

Worth noting: stereo cameras don't see textureless surfaces. Pointing at a blank wall produces almost no valid depth, because the matcher has nothing distinctive to align between the left and right images. This is fundamental to how stereo works — every pixel needs a unique-enough local pattern to be matched. Textured scenes (cluttered desks, foliage, brick walls, fabrics) work great. Smooth painted walls don't. Knowing where your cameras can and can't see is part of working with this technology.

## Stage 4: Multi-view fusion

This is where the three-camera investment pays off, in theory.

You have three depth maps, one per pair. Each is in *its own pair's* coordinate frame — specifically, the rectified left camera's frame for that pair. To fuse them, you need to express all the points in a single shared frame.

I picked CAM_A (the center camera) as the shared frame. Points from the (CAM_A, CAM_B) pair are already in CAM_A's frame after undoing rectification. Points from (CAM_C, CAM_A) need a single extrinsic transform. Points from (CAM_C, CAM_B) need to undo their pair-specific rectification, then apply the C→A extrinsic.

Concretely, for each pair's points:
1. Undo the rectification rotation: `p_orig_camera = R1.T @ p_rect`
2. If the pair's left camera is not CAM_A: apply the appropriate extrinsic: `p_A = R_extrinsic @ p_orig + T_extrinsic`

In numpy, with points stored as row vectors of shape `(N, 3)`:

```python
points_orig_left = points_3d_rect_left @ R1   # @ R1 == R1.T applied to row vectors
points_A = points_orig_left @ R_CA.T + T_CA.ravel()
```

A useful sanity check that confirmed my extrinsics were good: the C→A→B chain composition vs. the directly-measured C→B extrinsic agreed to within 0.81mm. That's an internal consistency check — if the chain disagreed by tens of millimeters, the extrinsic calibration would have been wrong, and fusion would never have worked.

After transforming, you concatenate all three pair-clouds and apply some cleanup:

- **Voxel downsampling** at 5mm: replaces clusters of points within 5mm cubes with a single representative point. Reduces redundancy and noise.
- **Statistical outlier removal**: rejects points whose distance to their nearest neighbors is unusually large. Removes flying outliers from bad matches.

Both are Open3D one-liners.

## The detour: rendering and what it teaches you

After implementing fusion, I rendered preview images of the resulting cloud from three angles: front (looking at the scene from the cameras' position), top (bird's eye), and side (looking across).

The front view looked roughly right but mirrored — the towel in the photo was on the left, but in the cloud render it was on the right. The side and top views looked dramatically wrong: streaks of points spraying out into space, scene structure barely recognizable.

I assumed the worst — that there was a fundamental bug in the coordinate transforms, that the calibration was somehow wrong, that fusion was broken. Spent an hour writing diagnostic scripts: per-pair renders, transform consistency checks, depth statistics.

The diagnostics all came back clean. Each pair's per-image extrinsic was internally consistent. Disparity values were sensible. Depth ranges were reasonable. The chain composition agreed to sub-millimeter precision.

The breakthrough came from loading the cloud in Open3D's *interactive* viewer with a coordinate-axis frame visible. The cloud looked perfectly correct — when viewed from along the +Z axis. From any other angle, the streaks reappeared.

Two separate issues were tangled together:

**1. Buggy offscreen rendering camera.** I had been telling Open3D's offscreen renderer to look at the cloud from the wrong direction. The "front" view was set up to render from behind the scene, looking outward — producing a mirrored render. Fixing the camera setup made the front view match reality (towel on the left, cabinet on the right).

**2. Real artifacts in the side view.** Even with correct rendering, the side view shows streaks. These are not bugs.

## Why side views of stereo clouds are streaky

The fundamental insight took me an embarrassingly long time to internalize:

**Stereo cameras produce 2.5D maps, not 3D models.**

A single camera pair sees the scene from a single direction. It assigns a depth value to each pixel — but depth values have errors of a few percent at the relevant distances, and disparity quantization (1 pixel of disparity is a different depth at different distances) produces additional discretization. When you look at the cloud from the *front* (the same angle the cameras saw), all those errors line up along the line of sight and are invisible — the points form a recognizable scene.

When you rotate to the side, those same errors become *spatial offsets* in the new view direction. A point that's "supposed to be at depth 1500mm" but actually got computed at 1480mm shows up correctly when viewed from the front (you can't tell from a single 2D projection) but visibly offset when viewed from the side. Multiply by tens of thousands of pixels and you get streaks.

This is not a bug. It is a property of the data. Three things make it worse:

- **Lower resolution** (we're at 1024×576): fewer pixels = less precise disparity = bigger streaks
- **Smaller block size** in matching: more per-pixel noise
- **Depth discontinuities**: at object edges, disparity must change abruptly. SGBM's smoothness penalty often "rounds the corner," producing transitional depth values that don't correspond to real geometry

The standard mitigation is **WLS filtering** — a post-process that smooths the disparity map while respecting image edges. It dramatically improves the front view (fills in textureless regions, smooths surfaces) and *somewhat* improves side views (less wild scatter). It does not eliminate the artifacts because those come from the fundamental information limitation, not the noise on top of it.

I tuned WLS for a while. lambda=8000, sigma=1.5 (defaults) over-smoothed across edges, dragging towel pixels into cabinet space. lambda=4000, sigma=1.0 was a better balance. lambda=2000 left more honest holes but at the cost of completeness. I settled on the middle option.

## What you can and can't conclude from a stereo cloud

This is the most important lesson from the whole detour:

**You can trust the front view.** Pixels in the rectified left image have known 3D positions. If you ask "what's the 3D position of this pixel I see in the image," the answer is reliable to within a few centimeters at typical distances.

**You cannot trust an arbitrary view of the cloud.** If you rotate to look from a direction the cameras never saw, you're asking for information that wasn't measured. The points displayed at those novel angles are projections of imperfect depth estimates from the original viewpoint, not actual measurements of the scene from the new angle.

This isn't a limitation of OpenCV or my calibration. It's the same limitation any stereo system has, including expensive industrial ones. To get a "real" 3D model — one that's correct from all viewpoints — you need either multiple physical viewpoints (e.g., a robot arm sweeping the cameras around the scene) or a different sensor modality (LiDAR, structured light, time-of-flight).

## What this means for the project

For the eventual blueberry-picking task, this matters less than it might seem.

The application workflow will be:
1. Detect berries in the 2D image (a YOLO model running on the rectified left view)
2. For each detected berry, look up the depth at its centroid
3. Project the centroid pixel + depth to a 3D position
4. Send that 3D position to the arm

None of these steps require a clean side view of the cloud. The whole pipeline operates on individual pixels of the rectified front view, where the data is good. The side-view artifacts live in regions of 3D space that the workflow never queries.

Eventually, when the rover circles the bush taking captures from multiple angles, those captures will *together* fill in the parts of 3D space that any single capture can't see. That's a Phase 3 problem — multi-pose multi-view photogrammetry — and it's a substantially different technique. For now, single-pose stereo is the right tool, and it works.

## Lessons from Step 3

**Stereo cameras produce 2.5D maps, not 3D models.** This is the most important sentence in this post.

**Verify rectification visually with horizontal scan lines.** It's a free correctness check that takes 30 seconds.

**Disparity range matters more than you think.** numDisparities is the parameter that decides whether you can see close objects. Tune it to your actual operating range.

**Ground-truth your depth with a tape measure.** Don't trust the calibration chain until you've verified it produces correct distances on something you can physically measure.

**Sanity-check your extrinsic chain composition.** If you have three cameras and three pairwise extrinsics, the chained transform from one to another should equal the directly-measured transform. Sub-millimeter agreement is achievable; tens of millimeters means something is wrong.

**Use Open3D's interactive viewer for debugging coordinate frames.** Offscreen renderers introduce their own coordinate conventions and can hide real bugs behind rendering bugs. Loading a cloud with axis frames in interactive mode and rotating freely is the fastest way to confirm what's actually in the data.

**WLS post-processing helps but doesn't fix everything.** Tune lambda and sigma to your scene; defaults are not always good. Trade off between filled-in surfaces (high lambda) and edge fidelity (low sigma).

**Look for a side view that "looks right" only when you've earned it.** If your scene has hard depth discontinuities (foreground objects in front of background), expect smearing across them. If your scene has smooth depth gradients (foliage, terrain), expect cleaner side views. The latter is closer to your eventual application.

## What's next

The perception pillar is done. The remaining Phase 1 work shifts from geometry to learning:

- Capture a dataset of blueberry images (200-500 images, varied lighting and angles)
- Label with Roboflow or CVAT
- Train YOLOv8n or v11n to detect berries
- Run on live webcam feeds, verify it identifies berries
- Combine: 2D detection from YOLO + depth lookup at the centroid = 3D berry position

That's the bridge from "we have working depth perception" to "we know where the berries are." From there, the project becomes about manipulation (Phase 2) and mobility (Phase 3).
