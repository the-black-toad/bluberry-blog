---
layout: post
title: "Building a Blueberry Picker, Phase 1 Retrospective"
date: 2026-05-26
---

# Building a Blueberry Picker, Phase 1 Retrospective

*May 26, 2026*

Phase 1 is done. Twenty-two tasks, a multi-camera perception stack, a trained detector, and a working 3D localization pipeline. This post covers what worked, what didn't, how the estimates held up, and whether Phase 2 is worth starting.

## What Got Built

The full Phase 1 deliverable is a pipeline that takes live frames from three USB webcams and outputs a list of 3D berry positions:

1. **Intrinsic calibration** — OpenCV checkerboard calibration for CAM_A (C920), CAM_B (C930e), CAM_C (C930e). Reprojection error <1px on all three.
2. **Extrinsic calibration** — pairwise calibration for all three stereo pairs (narrow_right: A–B, narrow_left: C–A, wide: A–C). Cameras mounted on an aluminum bar with ~100mm (narrow) and ~200mm (wide) baselines.
3. **Stereo rectification** — `stereoRectify` + precomputed remap tables per pair. Epipolar lines align to within 1–2px.
4. **Disparity and depth** — SGBM on the rectified pairs. Q matrix projection gives (x, y, z) in mm.
5. **Multi-view fusion** — point clouds from all three pairs merged into a single CAM_A frame using the extrinsic chain.
6. **YOLO detection** — YOLOv8n fine-tuned on 116 DSLR images / 9,818 labeled blueberry instances. mAP50=0.678, P=0.812, R=0.597.
7. **3D localization** — 2D centroid → disparity lookup → Q projection → extrinsic transform → cross-view clustering. Accuracy: ±5–10mm X/Y, ±2mm Z at ~350mm depth.
8. **Open3D validation** — interactive point cloud viewer with 25mm red spheres at each detected berry; confirmed sphere placement matches physical berry positions.

## What Worked Well

**Stable camera identification by USB path** was the right call early. OpenCV indices shift on replug; USB paths don't. Every script in the repo uses `cameras.py` to look up the right device, and nothing has broken due to index drift since that was locked in at step 1.

**Median patch depth sampling** solved a real problem. SGBM leaves holes around object edges — exactly where bounding box centroids tend to fall. A 7×7 patch median drops the hole rate enough that depth reads are reliable on real berry detections without needing to fill the disparity map.

**YOLO over HSV baseline** was the right call even for unripe berries. The HSV approach over-segments dense clusters badly; YOLO handles occlusion and scale variation much better. The 68% mAP50 result on DSLR-only data is acceptable given how different the training distribution is from the inference domain.

**Checkerboard during validation** turned out to be useful for an unintended reason: SGBM needs texture to compute disparity, and the checkerboard pattern behind the berries gave it dense, clean depth readings right where the detections landed. A useful trick for future lab validation.

**Threading for stereo pairs** gave a real speedup. Running SGBM on the two stereo pairs in a `ThreadPoolExecutor` keeps the FPS at ~4.2 rather than ~2.5. SGBM releases the GIL, so both pairs actually run in parallel.

## What Didn't Go as Planned

**Roboflow split configuration** was a silent gotcha. The first dataset export put all 348 images in `train/` — Roboflow requires explicit split assignment before generating a version, and there's no warning if you skip it. Also bit: `data.yaml` exports with `../train/images` paths that silently fail without manual editing. Both are easy fixes; both cost ~30 minutes each.

**CPU torch on GPU machine.** A `requirements.txt` generated on the laptop had `torch+cpu` pinned. Installing it on the GPU desktop silently ran training on CPU. Checking `torch.cuda.is_available()` before training is now on the mental checklist.

**Single-class model is a limitation.** The current model has one class: `blueberry`. That's correct for a detector but wrong for a picker — the arm should only target ripe berries. The ripe/unripe distinction is deferred to Phase 1.5 (whenever ripe season arrives), but it means Phase 2 integration will need to assume all detected berries are targets, or add a ripeness filter later.

**Recall at 60% will matter more later.** Right now a missed detection just means one fewer point in the list. Once an arm is attempting picks, a missed detection means a missed berry. The unripe-DSLR training set is not the distribution the arm will encounter. Plan for a retrain with ripe webcam images before Phase 2 field testing.

## Time Estimates vs. Actuals

The plan estimated Phase 1 at roughly 60–70 hours across 22 tasks. Actual is harder to measure (weekend sessions over several weeks), but the tasks that hit their estimates:

- Intrinsic calibration: on target
- Extrinsic calibration + stereo pipeline: on target
- YOLO training: faster than expected once the Roboflow split issue was resolved
- 3D localization: slightly over — the rectification-frame coordinate bug (forgetting to undo R1 before applying the cross-camera extrinsic) cost ~2 hours

The main underestimate was infrastructure overhead: environment setup across two machines, Roboflow account setup, dataset export/import friction. None of these show up as named tasks but collectively added maybe 4–5 hours.

## Go / No-Go for Phase 2

**Go.**

The localization output — a list of (x, y, z) positions in mm, stable to ±2mm in depth and ±10mm laterally, at ~4fps — is sufficient for a static arm to attempt picks. A 12mm-radius berry sitting at a known position with <10mm error is a tractable target.

The three questions for go/no-go were:

1. *Can the system detect blueberries reliably enough to be useful?* Yes. 68% mAP50 on out-of-domain data is a starting point, not a ceiling. The live demo on fresh-picked berries was convincing.
2. *Is the 3D position accurate enough for a picking arm?* Yes. ±10mm lateral, ±2mm depth at 350mm range. A gripper with any compliance at all can handle that error budget.
3. *Is the pipeline fast enough for real-time closed-loop control?* Probably. 4fps is slow for high-speed motion but fine for a slow static arm making one pick at a time.

Phase 2 is the arm: a servo-driven stick arm, a simple gripper, and a wrist-mounted camera for final approach. Budget estimate ~$457 for components. The Phase 1 output (3D centroid list) becomes the Phase 2 input (pick target).

## Seasonal Note

Blueberries won't be ripe until late June or July. Phase 2 mechanical build can happen in parallel with the season — arm design, servo selection, and forward kinematics don't require ripe berries. The retrain with ripe-class webcam data should be timed to coincide with Phase 2 integration testing.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
