---
layout: post
title: "Three Cameras, One Coordinate Frame"
date: 2026-05-06 21:13:00 -0400
categories: [Computer Vision, Setup]
tags: [opencv, calibration, extrinsics, stereo, multi-camera]
---

*Or: how a USB bandwidth detour, a mount built from drilled aluminum, and an evening of patient board-holding produced sub-millimeter spatial calibration.*

This is the second post in a series documenting a multi-camera point cloud project. The [first post](setup_and_calibration.md) covered environment setup and per-camera intrinsic calibration — figuring out each camera's optics. This post covers Step 2: extrinsic calibration. Now that we know what each camera does individually, we figure out where they sit relative to each other in 3D space.

The result: each pair of cameras' relative position measured to within a millimeter or two of the physical mount, with sub-half-pixel reprojection accuracy. The math is well-understood and OpenCV handles most of it. The interesting parts are the design decisions and the inevitable detours.

## What "extrinsic" means

Intrinsics are about each camera in isolation: focal length, distortion, principal point. They tell you how a 3D ray entering one camera maps to a 2D pixel coming out.

Extrinsics are about the relationships *between* cameras. For each pair, the extrinsic calibration produces a rotation matrix and a translation vector. Together these describe how the second camera's coordinate frame is positioned and oriented relative to the first.

If you have a rig with two cameras, extrinsic calibration tells you "camera B is 100mm to the right of camera A and rotated 0.5° clockwise." With three cameras, you describe pairwise relationships. The math gets richer with three because you have the option to do a global solve that's more accurate than three independent pairwise solves — but pairwise is the standard and it's what we did.

You need extrinsics to do anything 3D involving multiple cameras. Triangulating a point seen by both cameras requires knowing where each ray comes from in a shared world coordinate frame. Without extrinsics, each camera knows its own optics but has no idea where it is.

## The physical setup

Three USB webcams on a fixed mount. The decision tree was small but worth thinking through:

**Why a fixed mount and not flexible positioning?** Once you do extrinsic calibration, the cameras must not move. Period. A 1mm shift in any camera invalidates the calibration. Fixed mounts let you trust your numbers; flexible mounts mean recalibrating constantly.

**Why a flat line and not a V-shape?** A V-shape splays the cameras outward for wider total field of view. A flat line keeps all three pointing at the same scene with overlap that's good for stereo matching. For depth perception (the project goal), flat line wins. V-shape would have been right if I wanted peripheral situational awareness for navigation, but the rover this eventually mounts on can rotate to look around.

**Why this baseline?** 4 inches between adjacent cameras (so 8 inches between the outer pair). The baseline tradeoff: wider gives better depth precision at long range but worse close-range and larger occlusion zones. For a tabletop / close-range application (~30cm-1m operating distance), the math says baselines around 1-4 inches are appropriate. 4 inches is at the wide end of that band — a bit better far-range precision while still working close in.

**Why C920 in the middle?** The two C930e's at the ends form a wide-baseline pair with matched optics. Stereo matching works much better between cameras with identical lens parameters because features look the same in both. The C920's narrower field of view makes it natural as a center "verification" camera that overlaps with both ends.

**The mount itself:** drilled aluminum bar with three 1/4-20 holes at 0", 4", 8". Cameras mounted via their built-in clip bases using their tripod-thread sockets. Took about an hour to build with a hand drill, calipers, and tape measure. Cost ~$10 in parts.

## The bandwidth detour

This is where the day disappeared.

Three USB webcams streaming MJPEG at 1280×720 30fps comes out to about 75-105 Mbps total — well within USB 2.0's practical 280 Mbps ceiling. So that should work. It mostly does — you saw three streams in the first post — but in practice, the third camera was on the edge, sometimes failing to allocate bandwidth on connection.

The seemingly obvious fix: use USB 3.0. The laptop has Thunderbolt 3 ports (2018-era Dell Latitude 7390 2-in-1, JHL6540 controller). I bought a USB-C to USB-A 3.0 cable, plugged the hub through that into the Thunderbolt port, and...

```
/:  Bus 003.Port 002: Dev 002, Class=Hub, Driver=hub/4p, 480M
```

`480M`. USB 2.0 speeds. Same thing happened on the other Thunderbolt port. Same thing happened with a different cable. The hub kept negotiating USB 2.

The investigation went deep:
- BIOS was 9 years out of date — fwupd couldn't update it (no EFI System Partition on the Artix install)
- Tried `boltctl` to see if Thunderbolt authorization was the issue (it wasn't — the hub isn't a Thunderbolt device)
- Verified the cable was genuinely USB 3 (blue plastic, 5G rated)
- Confirmed via `lspci` that the Thunderbolt 3 controller exists and is detected

I don't actually know why USB 3 won't negotiate. Possibilities include: a BIOS setting I couldn't reach without booting the Dell USB-key BIOS update, a Linux kernel quirk with the JHL6540, a hub-firmware incompatibility. The hardware is capable; something in the chain isn't letting USB 3 through.

After 90 minutes of debugging, I gave up and chose a middle path: drop to **1024×576**. Same 16:9 aspect ratio as the original 1280×720 calibration, which means intrinsic calibrations scale cleanly by a factor of 0.8. Three cameras at this resolution fit comfortably in USB 2 bandwidth. Depth precision is ~25% worse than at 720p, which for tabletop work is the difference between "very good" and "slightly less good." The project still works.

The lesson: spending time fighting hardware quirks has steeply diminishing returns when there's a workable alternative. Every minute on USB 3 was a minute not spent on actual extrinsic calibration. The yak got partially shaved; I left.

## Scaling intrinsics to the new resolution

OpenCV intrinsics are in pixel units, so they scale linearly with image dimensions. Going from 1280×720 to 1024×576 (a uniform 0.8x scale) means every component of `K` scales by 0.8:

```python
K_scaled = K.copy()
K_scaled[0, 0] *= scale_x   # fx
K_scaled[1, 1] *= scale_y   # fy
K_scaled[0, 2] *= scale_x   # cx
K_scaled[1, 2] *= scale_y   # cy
```

Distortion coefficients (k1, k2, p1, p2, k3) are dimensionless — they describe lens curvature, not pixel-space measurements — and don't scale.

A subtle point: this scaling is mathematically exact only if the camera's downsampling is perfect. Real-world resizing introduces tiny artifacts (interpolation, anti-aliasing) that mean the scaled calibration is *almost* right but not perfectly. For best results you'd recalibrate at the operating resolution. For acceptable results you scale and move on. I scaled.

The scale-vs-recalibrate decision came down to time budget. Recalibrating three cameras is an evening of work. Scaling is 30 seconds. The accuracy difference is small enough not to matter for the project's goals. Easy choice.

## The simultaneous capture problem

Single-camera intrinsic calibration is easy to capture: hold the board, watch the live preview, press space, repeat 25 times. With three cameras, you need all three to capture *at the same moment* — same physical board pose, same instant in time. The board has to be visible to all three.

USB webcams don't hardware-sync. When you call `cap.read()` on three cameras sequentially, the frames you get are from slightly different moments — typically within ~30ms but not zero. For static scenes (you holding a board still), this is fine. For dynamic scenes, you'd need software trickery (timestamp matching, buffer management) or hardware-synced cameras.

The capture script:

```python
# Grab frames from all three at "the same time"
frames = {}
grays = {}
for label, cap in caps.items():
    ret, frame = cap.read()
    frames[label] = frame
    grays[label] = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)

# Detect corners in all three
detections = {}
for label, gray in grays.items():
    found, corners = cv2.findChessboardCorners(gray, PATTERN, None)
    detections[label] = (found, corners)

all_found = all(d[0] for d in detections.values())

# Only allow saving when ALL THREE detect the board
if key == ord(' ') and all_found:
    for label, frame in frames.items():
        cv2.imwrite(f"{SAVE_DIR}/{label}/triplet_{count:02d}.png", frame)
    count += 1
```

The script saves a "triplet" — three images, one per camera, captured nearly simultaneously. If any of the three failed to detect the board, the triplet is rejected. Partial captures aren't useful for stereo math.

Two design choices in this loop worth noting:

**Sequential reads, not parallel.** I could thread the `cap.read()` calls to grab frames truly simultaneously. I didn't, because for a static scene the ~30ms span doesn't matter, and threading adds complexity that might silently introduce bugs. Always start with the simpler approach.

**All-or-nothing triplets.** Only saving when all three detect simplifies the downstream calibration code — every triplet contributes equally to every pair. Alternative: save partial triplets and use whichever pairs of cameras have data for each pose. More flexible, more code, more bugs to debug. Not worth it at this scale.

## Capturing the triplets

This is harder than single-camera capture. The constraints:

**Where to stand:** about 1-1.5m in front of the rig. Closer and the outer cameras lose the board off the side; farther and the board gets too small for reliable corner detection. The "all three see the board" overlap zone is narrower than it feels.

**Pose variety, but bounded.** For intrinsics, big board tilts are great because they constrain the optics solve. For extrinsics, big tilts are bad because they often take the board out of one camera's view. So tilts of 15-30°, not 45°. Lateral movement across the overlap zone, distance variation within it, and gentle tilts in all directions.

**Hold completely still.** Three sequential `cap.read()` calls span ~30ms. Any motion during that window puts the board in a slightly different position for each camera, which corrupts the extrinsic measurement. Not "freeze for a moment" — really, completely, no-shake-no-drift still. Brace your elbows on something if needed.

**Watch the status bar.** The script tells you which cameras don't see the board. CAM_C consistently failing means you're too far left. CAM_B failing means too far right. Adjust until "ALL THREE DETECTED" flashes green frequently.

I captured 31 triplets in about 20 minutes. All 31 had successful corner detection on all three cameras — a higher success rate than I expected, mostly because the live status feedback let me reject "this pose isn't working" and adjust quickly.

## The stereo calibration math

OpenCV's `stereoCalibrate` does the heavy lifting. The signature is busy:

```python
rms, K1, dist1, K2, dist2, R, T, E, F = cv2.stereoCalibrate(
    objpoints, img_pts_left, img_pts_right,
    K_left, dist_left, K_right, dist_right,
    img_shape,
    flags=cv2.CALIB_FIX_INTRINSIC
)
```

`CALIB_FIX_INTRINSIC` is important: it tells OpenCV not to re-fit the intrinsic parameters. We already calibrated those individually with much more pose variety than we have here, and the intrinsic results were excellent. Letting `stereoCalibrate` re-fit them on this constrained dataset would be strictly worse. The flag locks the intrinsics and only solves for the relative pose between cameras.

Internally, the algorithm does roughly: for each triplet, project the known 3D board into both cameras using the intrinsics and a candidate pose, compare to the observed 2D corner positions, sum the squared error across all triplets, minimize jointly over the relative pose parameters (R and T). It's the same Levenberg-Marquardt machinery as intrinsic calibration, just applied to a different objective.

The output:
- **R**: 3×3 rotation matrix
- **T**: 3×1 translation vector, in millimeters (because object points were in millimeters)
- **E, F**: essential and fundamental matrices, useful for downstream stereo work

We do this three times for the three pairs: (CAM_C, CAM_A), (CAM_A, CAM_B), (CAM_C, CAM_B).

## The results

Here's where it got satisfying.

```
CAM_C_CAM_A: baseline 103.38 mm (4.07 in), rotation 2.47°, RMS 0.306 px
CAM_A_CAM_B: baseline 101.22 mm (3.98 in), rotation 2.99°, RMS 0.369 px
CAM_C_CAM_B: baseline 203.82 mm (8.02 in), rotation 3.09°, RMS 0.389 px
```

Going through these:

**Baselines match the design within fractions of a millimeter.** Designed for 4" and 8"; got 4.07", 3.98", 8.02". That's ~1mm error on the 4" pairs and 0.5mm on the 8" wide pair. Better than I expected from a hand-drilled bar.

**Translation vectors are clean.** All three pairs show large X components and tiny Y/Z (within 1-2mm). The cameras are essentially coplanar and arranged horizontally, exactly as designed. If the Y values had been 10mm instead of <1mm, that would have meant one camera was mounted significantly higher than the others.

**Rotation angles are 2.5-3°.** This was unexpected at first. I designed for "all parallel"; got "each pair is tilted by ~3° relative to its neighbor." After thinking about it, the explanation became obvious: the Logitech clip-style camera bases have a tilt joint that allows a few degrees of free play even when "locked." The cameras can rotate slightly within their mounts. Three cameras × few degrees of freedom each = small per-pair rotations, which is exactly what we measured.

This isn't a problem. Stereo math handles rotated cameras transparently — the calibration measures the rotation precisely and downstream depth computation accounts for it. The implication is just that the calibration is specific to *this rig in this exact configuration*. Bumping a camera's tilt invalidates the calibration; you'd recapture and redo.

**RMS errors of 0.31-0.39 px.** Excellent. For reference, intrinsic RMS was ~0.15 px; stereo RMS being roughly 2x that is normal because stereo calibration also has to handle pose estimation between cameras and the small temporal misalignment between the three sequential captures.

**Triangle inequality check:** C-A baseline (103.4mm) plus A-B baseline (101.2mm) equals 204.6mm. C-B baseline (203.8mm). The discrepancy is 0.8mm, or 0.4%. For three nominally collinear cameras, this is essentially perfect — they really are on a straight line.

## What the numbers mean for the project

You can take any pixel from any camera and compute the 3D ray it represents. With pixels from two cameras seeing the same point, you can triangulate the 3D position of that point. With three cameras seeing the same point, you can triangulate with redundancy and reject outliers. This is the foundation for stereo depth, point cloud generation, and any downstream 3D vision work.

For your specific rig at 1024×576 in the tabletop range:
- Depth precision at 0.5m: ~5-7mm using the wide-baseline pair
- Depth precision at 1.0m: ~15-25mm using the wide-baseline pair
- Close-range (under 0.5m): the narrow-baseline pairs take over because the wide pair has too much disparity

For tabletop manipulation and obstacle detection, this is plenty.

## Lessons from Step 2

**Build the mount before calibrating; never move cameras during the project.** Calibration captures the rig in a specific physical state. Movement invalidates calibration. Plan accordingly.

**Use `CALIB_FIX_INTRINSIC` for stereo calibration.** Trust the intrinsics you measured carefully, don't let stereoCalibrate re-fit them on noisier multi-camera data.

**Capture triplets where all three cameras see the board.** Partial captures complicate the math without adding useful information. Reject them at capture time.

**Sanity-check the geometry.** Translation vectors should be dominantly along one axis (horizontal, for a flat-line rig). Y and Z components should be tiny. If they're not, your physical mounting doesn't match your design.

**Triangle inequality is a free consistency check.** For three collinear cameras, the sum of two adjacent baselines should equal the wide baseline. Discrepancy reveals geometric problems the per-pair RMS won't catch.

**The Logitech clip bases have rotational play.** If you want sub-degree mount precision, bypass the clip bases and bolt camera bodies directly to the mount via their tripod threads. If you don't care about absolute orientation (because the calibration measures whatever you have), the clip bases are fine.

**Yak-shaving has limits.** The USB 3 detour cost me ~90 minutes. The alternative path (drop resolution, scale intrinsics) cost ~5 minutes and produced a working setup. The actual project is more important than any single technical perfection within it. If you find yourself two hours into debugging infrastructure, ask whether there's a workable alternative.

## What's next

With intrinsics and extrinsics done, the geometric foundation is complete. The next steps:

**Stereo rectification:** for each pair, warp the images so corresponding points lie on the same horizontal scanline. This makes stereo matching tractable (1D search instead of 2D). OpenCV's `stereoRectify` and `initUndistortRectifyMap` handle this.

**Disparity computation:** for each pixel in the left rectified image, find the corresponding pixel in the right and compute their horizontal offset. Standard algorithm: SGBM (Semi-Global Block Matching), tunable, included in OpenCV.

**Depth from disparity:** the formula is `depth = (baseline × fx) / disparity`. Apply per-pixel.

**Point cloud generation:** for each pixel with valid depth, compute the 3D position. Visualize with Open3D.

After that, the actual application work begins — what does the rover need to *do* with this depth perception? Obstacle detection, scene understanding, object recognition, path planning. Different problems, different code, but the geometric foundation underneath stays the same.
