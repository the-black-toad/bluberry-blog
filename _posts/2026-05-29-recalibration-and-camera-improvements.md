---
layout: post
title: "Recalibration, Corner Flips, and Better Camera Tooling"
date: 2026-05-29
categories: [Blueberry Picker]
tags: [calibration, stereo, opencv]
---

![](https://pub-7dccfb8281d647ea908c15e9e1c6a9cb.r2.dev/blueberry-blog-images/images/img_5368.jpg)

A new camera mount meant the old extrinsic calibration was invalid. What should have been a 20-minute recapture-and-recalibrate turned into a debugging session that improved the calibration pipeline significantly. This post covers the issues hit along the way and the fixes that came out of it.

## What Changed

The camera rig was remounted on a new bar. The cameras moved to different USB hub ports, the physical left-to-right order changed, and CAM_C had a small downward tilt that the old mount didn't have. All three required fixes before recalibrating.

## OpenCV Headless vs. GUI

The first issue was immediate: `cv2.imshow` crashed with `The function is not implemented` even though the display was available. The root cause was a conflicting installation — both `opencv-python` and `opencv-python-headless` were installed in the venv, and the headless build was winning. The fix:

```bash
python -m pip uninstall opencv-python opencv-python-headless opencv-contrib-python -y
python -m pip install opencv-contrib-python
```

Note the `python -m pip` — the venv's `pip` binary was resolving to the system pip, which refused to run due to Arch's externally-managed-environment restriction.

## Camera Paths and Physical Layout

`cameras.py` maps labels (CAM_A, CAM_B, CAM_C) to stable USB paths. After the remount, `v4l2-ctl --list-devices` showed the C920 (CAM_A) had moved from port `9.1` to `9.3`, and the physical left-to-right order changed from C | A | B to B | A | C.

Two problems with the old approach:

1. USB paths were hardcoded in multiple places — updating them meant touching `cameras.py`, `view_all_cameras.py`, `capture_extrinsic.py`, and anything else with a display layout.
2. There was no tooling to help identify which camera is which after a remount.

**Fix:** Added `PHYSICAL_ORDER` to `cameras.py` as a single source of truth, and updated all scripts to import it instead of hardcoding the order. Now changing the layout is a one-line edit in one file.

```python
# cameras.py
PHYSICAL_ORDER = ["CAM_B", "CAM_A", "CAM_C"]  # left to right
```

**Also added:** running `python scripts/cameras.py` launches an interactive wizard. It auto-identifies the C920 (unique model) as CAM_A, then shows each C930e live and asks you to press B or C to label it. It then rewrites `CAMERA_PATHS` and `PHYSICAL_ORDER` in-place. Future remounts won't require manually cross-referencing `v4l2-ctl` output.

## CAM_C Tilt

The initial recalibration showed 2.1° rotation on CAM_C across both pairs involving it (C-A and C-B), while the A-B pair was only 0.79°. In camera coordinates, CAM_C's optical axis was pointing about 1.7° downward relative to CAM_A. The fix was physical: loosen the mount screw, nudge the lens up a few degrees, retighten.

After adjustment, all three pairs were under 0.87° rotation — an improvement over the original mount.

## Corner Flip Issue

After adjusting the mount and recapturing, the calibration results got *worse*, not better:

```
CAM_C_CAM_A:  1.33 px   (was 0.85 px)
CAM_A_CAM_B:  1.36 px   (was 0.69 px)
CAM_C_CAM_B:  0.86 px   (unchanged)
```

C-B was unaffected but both narrow pairs jumped. The pattern isolated the problem to CAM_A's detections: C-A and A-B both involve CAM_A; C-B does not.

The cause was **corner ordering flips**. For non-square boards (8×6), `findChessboardCorners` returns corners in row-major order, but at certain orientations it can flip which end is "corner 0." The first recapture set used mostly flat-on boards so flips rarely happened. The new set had more varied tilts, and some CAM_A captures were detecting from the opposite end.

**First fix attempt:** normalize by requiring corner[0] to have the smallest y coordinate. This improved things but failed for left/right tilts, where the x-axis is the primary source of ambiguity.

**Second attempt (x+y):** require corner[0] to have the smallest x+y sum. Better, but still failed at extreme tilts where the board occupies an unusual perspective in one camera.

**Final fix:** cross-camera majority vote. All three cameras see the same physical board, so the direction of the first row (corner[0] → corner[1] in x) must be consistent. If two cameras agree and one disagrees, flip the outlier:

```python
row_dirs = {cam: np.sign(c[1][0][0] - c[0][0][0])
            for cam, c in triplet_corners.items()}
majority = np.sign(sum(row_dirs.values())) or 1
for cam, d in row_dirs.items():
    if d != majority:
        triplet_corners[cam] = triplet_corners[cam][::-1]
```

This is robust because it uses physical consistency across cameras rather than assumptions about how the board appears in any single image. It handles all tilt directions without any heuristic thresholds.

## Per-Triplet Outlier Rejection

Even with the flip fix, some RMS remained from noisy captures — blurry frames or corner detections that technically succeeded but with poor subpixel accuracy. Added a per-triplet quality filter using `solvePnP`:

```python
def per_image_error(objp, imgp, K, dist):
    _, rvec, tvec = cv2.solvePnP(objp, imgp, K, dist)
    proj, _ = cv2.projectPoints(objp, rvec, tvec, K, dist)
    return float(np.sqrt(np.mean((proj - imgp) ** 2)))
```

For each triplet, compute the single-camera reprojection error per image. Reject any triplet where any camera exceeds 1.0px. On the 40-capture set this dropped 5 outliers (all with one camera >1.4px while the other two were under 0.2px — clear flip or blur events) and kept 31 clean ones.

## Final Results

```
Pair        RMS (original mount)    RMS (new mount)
C-A         0.31 px                 0.34 px
A-B         0.37 px                 0.41 px
C-B         0.39 px                 0.30 px
```

Better than the original mount on C-B, within noise on the other two. All rotations under 0.87°. Triangle inequality discrepancy 0.4% (cameras nearly collinear). Valid ROIs on all three rectified pairs are essentially full-frame.

## What Was Worth It

The calibration result is marginally better than before, but the real gains were in the tooling:

- `PHYSICAL_ORDER` in `cameras.py` means display layout is maintained in one place
- The setup wizard means future remounts don't require manual USB path detective work
- Cross-camera majority vote is a more principled corner ordering fix than any single-image heuristic
- Per-triplet outlier rejection with solvePnP is now a standard part of the calibration pipeline

The corner flip issue in particular is worth knowing about for anyone using non-square chessboards: varied capture angles are important for calibration quality, but they also increase the probability of flip events. The y-only or x+y normalization approaches that appear in most tutorials are fragile at extreme tilts. Cross-camera consistency is the right fix.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
