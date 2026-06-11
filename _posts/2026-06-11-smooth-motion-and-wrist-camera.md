---
layout: post
title: "Smooth Arm Motion and Wrist Camera Calibration"
date: 2026-06-11
categories: [Blueberry Picker]
tags: [phase-2, so-101, motion, kinematics, camera, calibration, wrist-cam]
---

Two things today: the arm stopped moving like a nervous robot and a wrist camera got calibrated. Both are prerequisites for picking.

## The Jerkiness Problem

The arm worked. It moved where it was told, to within ~20mm. But every move had a mechanical snap to it — the kind of motion that would knock a berry off a branch before the gripper ever got near it. For picking, that's a problem.

The cause was immediate to diagnose once looked at. The interpolation loop in `_interpolate()` and `move_to()` was computing position as:

```python
t = i / steps
pos = start + t * (target - start)
```

Linear interpolation means constant velocity. The arm snapped from stationary to full speed at `t=0` and stopped dead at `t=1`. No ramp-up, no ramp-down. The servos' internal PD controllers were fighting sudden target changes at every step, making it worse.

## Quintic Smoothstep

The fix is a standard one in robotics: replace the linear `t` parameter with a smoothstep profile. The quintic variant:

```python
t_s = t**3 * (6*t**2 - 15*t + 10)
pos = start + t_s * (target - start)
```

This profiles velocity as a smooth bell curve — zero at the start, peak at midpoint, zero at the end. Crucially, both velocity *and* acceleration are zero at the endpoints, which eliminates the jerk discontinuity entirely. The arm now accelerates smoothly, moves at full speed through the midpoint, and decelerates to a gentle stop. Night and day difference on the first test.

## Timing Jitter

The second issue was subtler. Python's `time.sleep()` on Linux drifts by ±5ms per call. At 20Hz (50ms per step), that's ±10% jitter per step — steps aren't evenly spaced in time even though the position profile is mathematically smooth. The velocity profile was correct on paper but ragged in execution.

The fix: instead of sleeping a fixed interval, compute the target wall-clock time for each step and sleep until it arrives:

```python
t_start = time.perf_counter()
for i in range(steps + 1):
    ...compute and send...
    deadline = t_start + (i + 1) / hz
    sleep = deadline - time.perf_counter()
    if sleep > 0:
        time.sleep(sleep)
```

Drift no longer accumulates — each step self-corrects. Any execution overhead from the previous step gets absorbed by a shorter sleep on the next one.

## Higher Control Rate

While here, the default control rate was bumped from 20Hz to 100Hz. The Feetech serial bus at 1Mbps handles 6-joint command packets in ~1.5ms, so 100Hz (10ms per cycle) is well within budget. At 100Hz over a 6-second move there are now 600 interpolation steps instead of 120. Combined with the smoothstep profile, the arm moves with a quality that's genuinely surprising for hobby servos.

Both `arm_control.py` and `go_home.py` were updated. The inline interpolation loop that had been duplicated in `move_to()` was also consolidated to call `_interpolate()` directly, so there's one code path for all moves.

## Wrist Camera

Phase 2 needs a close-up view of the berry during the final approach. The plan from the start was a wrist-mounted camera — a small USB module that rides on the arm and feeds into the pick logic.

The hardware choice: **ELP USB Camera Module, 1MP, standard 32×32mm PCB**. This matches the official SO-101 wrist camera mount from TheRobotStudio (`SO101_Wrist_Cam_Hex-Nut_Mount_32x32_UVC_Module`), which uses the hex-nut recesses already built into the SO-101 wrist design. Four M2 screws, no disassembly required.

The hex-nut mount was chosen over the integrated variant (which replaces the wrist roll component entirely) specifically because the arm is already assembled and calibrated. Disassembling the wrist would mean reassembly and likely recalibration. The add-on hex-nut mount avoids all of that.

One fitment issue: the ELP board camera has a USB connector that overhangs one edge of the PCB. The official mount's cavity is exactly 32×32mm with no clearance for the connector. A few minutes with a Dremel cut a slot for it. Mount is solid.

The camera mounted upside-down due to cable routing. This doesn't affect calibration. It will need a flip in the inference pipeline, which is a one-liner.

## Identifying the Camera

The existing `scripts/cameras.py` uses USB serial numbers for stable device identification across reboots. Running a scan after plugging in the ELP:

```
/dev/video8  2020032501  HD_USB_Camera
/dev/video6  5B3E41DF    HD_Pro_Webcam_C920
/dev/video2  815319DE    Logitech_Webcam_C930e
/dev/video4  A1C119DE    Logitech_Webcam_C930e
```

The ELP registered as `HD_USB_Camera` with serial `2020032501`. Added to `CAMERA_SERIALS` as `CAM_W`:

```python
"CAM_W": "2020032501", # ELP 1MP (wrist)
```

## Intrinsic Calibration

Same checkerboard procedure used for the rig cameras (8×6 inner corners, 24.95mm squares), but at closer range — 6 to 16 inches — since the wrist cam will be operating much closer to targets than the overhead rig.

A small calibration helper script (`arm/calibration_pose.py`) positions the arm for capture: from SAFE_HOME it tilts wrist_flex by -50° to bring the camera to a level forward-facing view, then holds position while the operator runs the capture script.

32 frames captured. First calibration run came back at 1.22px RMS — acceptable but not great. Looking at per-image errors, 7 frames were outliers (0.17–0.53px vs. the rest at <0.05px). Deleted those, reran on 25 frames:

```
RMS reprojection error: 0.237 px   ← excellent

K:
  fx = 663.7    fy = 666.7    (ratio: 0.9956 — nearly square pixels)
  cx = 393.6    cy = 309.4    (offset from center: -6.4px, +9.4px)

Distortion:
  k1=-0.463  k2=0.284  p1=-0.0005  p2=-0.0009  k3=-0.112
```

The distortion coefficients are worth noting: k1 is -0.46, which is a fairly strong barrel distortion for a 1MP board camera. Not surprising — small cheap lenses tend toward this. It'll need to be properly undistorted in the inference pipeline, not ignored.

Calibration saved to `calibration/results/calib_CAM_W.npz`.

## What's Next

Two things remain before the wrist camera is useful:

1. **Hand-eye calibration** — determine the fixed transform from the camera frame to the gripper frame. This is what lets a pixel coordinate get converted into an arm-space 3D position. Without it the camera is just a pretty picture.

2. **Integration** — wire `CAM_W` into the pick sequence: rig cameras for coarse 3D berry position, arm moves to a standoff (~50mm), wrist cam refines the exact position, final approach and grasp.

The arm moves smoothly. The wrist camera is calibrated. The next session closes the loop.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — stereo camera rig + YOLO detection + SO-101 arm for autonomous blueberry picking.*
