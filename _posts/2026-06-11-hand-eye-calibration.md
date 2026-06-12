---
layout: post
title: "Hand-Eye Calibration for the Wrist Camera"
date: 2026-06-11
categories: [Blueberry Picker]
tags: [phase-2, so-101, camera, calibration, hand-eye, wrist-cam, opencv]
---

The wrist camera is now geometrically tied to the arm. A point the camera sees in 3D can be converted to arm coordinates. This is the piece that makes closed-loop picking possible.

## What Hand-Eye Calibration Is

The wrist camera rides on the arm and moves with it. The arm's FK gives the gripper's pose in the base frame at any configuration. Hand-eye calibration finds the fixed transform between the camera frame and the gripper frame — the rigid offset that never changes regardless of where the arm is pointing.

With that transform in hand, a camera-space 3D point becomes:

```
T_cam2base = T_gripper2base @ T_cam2gripper
p_base = T_cam2base @ p_cam
```

That's the chain from pixel to arm coordinate. Every pick will use it.

## Capture Setup

The checkerboard (8×6 inner corners, 24.95mm squares) sits fixed on the desk, about 12 inches from the gripper in the home position. The arm moves through 23 joint-space poses, the wrist camera observes the board at each, and `solvePnP` computes the board-to-camera transform.

Poses were designed in joint space rather than via IK. The base configuration comes from the calibration pose (SAFE_HOME with wrist_flex tilted forward 50°), then varied by sweeping shoulder_pan (±12°), wrist_roll (±50°), and two arm height variants — one slightly extended and one raised with wrist_flex compensating to keep the camera looking at the board.

The auto-capture loop detects the board with `findChessboardCorners`, requires 8 consecutive stable frames, then captures and advances to the next pose automatically. No keyboard needed. A timeout of 8 seconds skips poses where the board isn't visible.

One non-obvious detail: the camera is mounted upside-down for cable routing reasons. The intrinsic calibration was done on native (upside-down) frames. For `solvePnP` to be consistent with that calibration, the raw frame has to be passed to detection — the flip for display is applied to the preview only, after detection.

## Bugs Fixed Along the Way

**Unit mismatch.** ikpy's FK returns meters. `solvePnP` with board points in millimeters returns millimeters. Passing both to `calibrateHandEye` without aligning units produces translations in the hundreds of kilometers. Fix: multiply FK translation by 1000 before saving each pose.

**Display scaling.** The solver output was being multiplied by 1000 a second time for display, making the result look like 119 km instead of 119 mm. The result saved to disk was correct; only the printout and JSON were wrong.

**V4L2 stream stall.** Covered in the previous post but worth repeating: the camera must be opened first, and the read loop must run continuously. Any blocking gap longer than ~5 seconds stalls the V4L2 buffer and produces black frames. The capture script opens the camera, starts reading immediately, and only calls blocking arm operations from within the read loop.

**Servo torque timeout.** While waiting for user input between poses, the Feetech servos were timing out and going slightly limp. When the next move started, `_interpolate` read the drifted position as the start, causing a small jump. Fixed with a 500ms heartbeat: `send_action` is called every 0.5s during the wait loop to keep the servos locked. The auto-capture redesign made this largely moot since there's no manual wait anymore.

## Result

18 poses captured, TSAI method:

```
R_cam2gripper:
[[-0.3268  0.0909 -0.9407]
 [-0.6025  0.7468  0.2815]
 [ 0.7281  0.6588 -0.1893]]

t_cam2gripper: [-103.9, -27.6, +51.6] mm
|t| = 119.2 mm
det(R) = 1.000000
```

The 119mm translation is the distance from the gripper frame origin to the camera. L_WRIST in the arm model is 165mm (wrist_flex joint to gripper tip), so the camera sitting 119mm from the gripper frame is geometrically consistent with the ELP mount position on the wrist link.

The large off-diagonal values in R are expected — the camera is mounted upside-down and at a skew angle relative to the gripper's coordinate axes.

## What's Next

The calibration chain is complete: intrinsics for all four cameras (three rig + one wrist), stereo calibration for the rig pairs, and now hand-eye for the wrist. The next step is wiring this into the pick sequence — rig cameras locate a berry in 3D, the arm moves to a standoff, the wrist camera refines the exact position, then the final approach and grasp.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry)*
