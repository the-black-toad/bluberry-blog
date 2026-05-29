---
layout: post
title: "Setting Up a Multi-Camera Computer Vision Project from Scratch"
date: 2026-05-05 10:15:00 -0400
---

# Setting Up a Multi-Camera Computer Vision Project from Scratch

*Or: how three webcams, an Arch Linux laptop, and an evening of debugging produced sub-pixel calibration accuracy.*

This is a writeup of the first two steps of a multi-camera point cloud project: getting a Python computer vision environment running, then calibrating each camera's intrinsic parameters. The goal of this post is partly to document what I did, but mostly to capture the *judgment calls* that aren't obvious from a tutorial — the moments where the path branches and you have to pick.

If you're just looking for code, jump to the script blocks. If you're trying to understand why things are the way they are, read top to bottom.

## The hardware

- **Laptop**: Dell-ish thing with an Intel UHD Graphics 620, running Arch Linux. No discrete GPU.
- **Cameras**: Three USB webcams — one Logitech C920 and two C930e, all connected through a powered USB hub.
- **Desktop**: A separate machine with a real GPU, accessible over SSH. This won't matter until much later, when training neural networks becomes the bottleneck.

The plan is: develop on the laptop, where the cameras live; punt heavy training jobs to the desktop when the time comes.

## Step 0: Environment Setup

### The Python version trap

The first thing I tried was a plain `python -m venv`. Arch ships the latest stable Python with no apologies, which at the time of writing meant Python 3.14. This was the first land mine.

The Python ecosystem trails Python releases by a few months. Major libraries — Open3D in particular — don't ship wheels for the newest interpreter for a while after release. Trying to `pip install open3d` on Python 3.14 produced:

```
ERROR: Could not find a version that satisfies the requirement open3d
ERROR: No matching distribution found for open3d
```

This is a well-known Arch friction point. The standard fix is to install an older Python alongside the system one. I tried `sudo pacman -S python312`, which failed because Arch's official repos only carry the current Python.

The right tool for this job turned out to be **uv** — a Python version and package manager that downloads standalone Python builds on demand. One install, and any version is available:

```bash
sudo pacman -S uv
uv venv cv_env --python 3.12
source cv_env/bin/activate
```

uv pulls Python 3.12 from a separate distribution channel (`python-build-standalone`) and stashes it in `~/.local/share/uv/python/`. Completely independent of pacman. When Arch updates the system Python next month, this venv won't break.

Worth knowing: `pyenv` does the same thing, and the AUR has packages like `python312` that build from source. uv is the fastest of the three and the cleanest to set up.

### The package list

```bash
uv pip install opencv-python opencv-contrib-python numpy open3d matplotlib
uv pip install torch torchvision
uv pip install ultralytics scikit-learn pandas tqdm jupyter ipykernel
```

A note on `opencv-python` vs `opencv-contrib-python`: install both in the same command. The `contrib` package adds modules like ArUco markers and SIFT that aren't in the base. If you install them separately at different times, you can end up with conflicting installs that import without error but segfault at runtime. Don't do this.

### The pip-vs-uv pip gotcha

After installing, I tried `pip freeze > requirements.txt` and got back a list of *system* packages — `nftables`, `arandr`, `pulsemixer`, `dbus-python`. None of my project's dependencies. What?

Diagnosis with `which`:

```
$ which python
/home/me/projects/bluberry/cv_env/bin/python   ✓ correct

$ which pip
/usr/bin/pip                                    ✗ system pip, not the venv's
```

This is a classic uv pitfall: uv installs packages into the venv but doesn't install `pip` itself there by default. So `python` correctly resolves to the venv, but `pip` falls through to the system one and reports the system's installed packages.

Two fixes:

```bash
# Use the explicit module form - always uses the active Python
python -m pip freeze > requirements.txt

# Or just use uv
uv pip freeze > requirements.txt
```

I switched to `uv pip` for everything after this. It's faster anyway.

### The CUDA build I didn't need

`pip install torch` defaults to whatever Python wheels are most readily available. On a Linux system with no GPU, this still installed a CUDA build of PyTorch — about 2GB of accelerator code I couldn't use. I confirmed by running:

```python
import torch
print(torch.__version__)             # 2.11.0+cu130
print(torch.cuda.is_available())     # False
```

The `+cu130` suffix is the giveaway. To check actual GPU hardware:

```bash
lspci | grep -iE 'vga|3d|display'
# 00:02.0 VGA compatible controller: Intel Corporation Kaby Lake-R GT2 [UHD Graphics 620]
```

Integrated graphics only. So I swapped to the CPU build:

```bash
uv pip uninstall torch torchvision
uv pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
```

This dropped install size by ~1.5GB and removed runtime warnings about missing CUDA libraries.

The bigger question: is CPU PyTorch good enough for this project? Mostly yes. Calibration is CPU-bound regardless. Pretrained model inference (YOLOv8 nano, for instance) runs at 5-15 fps on a Kaby Lake CPU — fine for occasional detection. The only thing CPU PyTorch is bad at is *training*, and that's exactly what the desktop is for.

### The git setup that almost went wrong

Before any project files, I made `.gitignore`. This matters because:

```
cv_env/                # 800MB venv
calibration/captures/  # raw PNGs, hundreds of MB
*.pt                   # model weights
runs/                  # ultralytics training output
```

If you don't gitignore the venv before your first commit, you'll have hundreds of megabytes of platform-specific binaries in your repo history that are useless to anyone (including future-you on a different machine).

I made one mistake here: I created the file as `.gitingore` (typo), and git silently ignored its contents because the filename was wrong. Caught it before committing only because I was paying attention to `git status` showing `cv_env/` as untracked when it should have been ignored. Easy fix once spotted, but a good reminder to actually verify your gitignore is working with:

```bash
git status --ignored | grep cv_env
# Should show cv_env/ under "Ignored files"
```

Test it before you commit, not after.

### The laptop/desktop split, planned upfront

Since I knew I'd eventually want to train models on the desktop, I structured `requirements.txt` to *not* pin PyTorch. Stripped the torch lines out:

```bash
grep -v -iE '^(torch|torchvision|torchaudio)==' requirements.txt > requirements.tmp
mv requirements.tmp requirements.txt
```

And documented the per-machine install in the README:

```
Laptop (CPU):
  uv pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu

Desktop (CUDA):
  uv pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
```

This means anyone (including future-me on the desktop) running `uv pip install -r requirements.txt` will get every dependency *except* PyTorch, then install the right PyTorch build for their hardware. Slightly more setup steps, much more portable.

## Step 1: Intrinsic Calibration

### What "intrinsic" means and why we need it

Every camera has a model. The simplest version: there's a sensor (a grid of pixels) and a lens (which bends incoming light). The intrinsic parameters describe the relationship between a 3D ray entering the camera and the 2D pixel coordinate where it lands.

The standard pinhole model has:
- **Focal length** (`fx`, `fy`): how "zoomed in" the camera is, in pixel units
- **Principal point** (`cx`, `cy`): where the optical axis hits the sensor, ideally the image center
- **Distortion coefficients** (`k1`, `k2`, `p1`, `p2`, `k3`): how much the lens bends straight lines into curves

Without these numbers, you cannot do any 3D math. Triangulating a point seen by two cameras requires knowing exactly which 3D ray each pixel corresponds to, and that's exactly what intrinsics encode.

The numbers are camera-specific. Same model webcam, different physical units, different intrinsics. You calibrate each camera you're going to use.

### The checkerboard

Camera calibration works by showing the camera something whose 3D geometry you know perfectly. A printed checkerboard is the standard choice because:

1. The corners are easy for software to detect
2. The corners lie in a perfectly known grid in the board's local coordinate frame
3. The board is flat, which constrains the geometry nicely

I downloaded an A4 PDF with 8×6 inner corners (9×7 squares) at a nominal 25mm square size. The "inner corners" thing matters: OpenCV counts the points where four squares meet, not the squares themselves. A pattern with 9 squares across has 8 inner-corner columns. Getting this wrong means corner detection fails silently.

Two non-obvious tips for the printed board:

**Disable any "fit to page" scaling in your printer dialog.** If the printer rescales by even 2%, your board's true square size doesn't match the advertised one and your calibration is off by a constant scale factor.

**Measure the squares with calipers** after printing. Don't trust the label. Mine came out at 24.95mm, not 25.00mm. That precision matters when distances start showing up in millimeters.

I mounted the printed sheet on a flat board (foam board with adhesive is the classic move; cardboard works but warps). The mount needs to be rigid enough that the paper doesn't bow during handling. A bowed board produces small but real calibration errors that look like bad luck.

### Identifying three identical-looking webcams

Three USB webcams plugged into a hub all show up in Linux as `/dev/video0`, `/dev/video1`, etc. The numbering changes across reboots and replug events. Two C930e's look physically identical. This is a recipe for mixing them up and using the wrong calibration file for the wrong camera.

The setup ritual I settled on:

1. **List the devices**:
   ```bash
   v4l2-ctl --list-devices
   ```
   Output includes USB topology paths like `usb-0000:00:14.0-9.1`. These paths are *stable* — they don't change across reboots, only across cable swaps.

2. **Find OpenCV's index for each device** by probing with a small Python script. OpenCV indices don't directly correspond to `/dev/videoN` numbers; OpenCV typically skips the metadata-only video nodes, so even-numbered indices are the real cameras.

3. **Visually identify which physical camera is at which index** by waving at each one in turn while a live preview is open.

4. **Stick a piece of tape on each camera** with a label: `CAM_A`, `CAM_B`, `CAM_C`.

5. **Document the mapping** in the README, including USB paths so you can recover it after a reboot:

   | Label | Model | USB path             | OpenCV idx |
   |-------|-------|----------------------|------------|
   | CAM_A | C920  | usb-0000:00:14.0-9.1 | 2          |
   | CAM_B | C930e | usb-0000:00:14.0-9.2 | 4          |
   | CAM_C | C930e | usb-0000:00:14.0-9.3 | 6          |

This took about 15 minutes and saved hours of confusion downstream.

### The capture script

The capture loop:

```python
import cv2, os, sys

PATTERN = (8, 6)
TARGET_W, TARGET_H = 1280, 720

CAMERA_ID = sys.argv[1]
DEVICE_INDEX = int(sys.argv[2])
SAVE_DIR = f"calibration/captures/{CAMERA_ID}"
os.makedirs(SAVE_DIR, exist_ok=True)

cap = cv2.VideoCapture(DEVICE_INDEX, cv2.CAP_V4L2)

# MJPEG codec - critical for USB 2.0 bandwidth with three cameras
cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
cap.set(cv2.CAP_PROP_FRAME_WIDTH, TARGET_W)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, TARGET_H)

# Disable autofocus - focus drift mid-session shifts focal length
cap.set(cv2.CAP_PROP_AUTOFOCUS, 0)

count = 0
while True:
    ret, frame = cap.read()
    if not ret: continue

    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    found, corners = cv2.findChessboardCorners(gray, PATTERN, None)

    preview = frame.copy()
    if found:
        cv2.drawChessboardCorners(preview, PATTERN, corners, found)

    cv2.imshow(f"Calib - {CAMERA_ID}", preview)
    key = cv2.waitKey(1) & 0xFF
    if key == ord(' ') and found:
        cv2.imwrite(f"{SAVE_DIR}/img_{count:02d}.png", frame)
        count += 1
    elif key == ord('q'):
        break
```

A few non-obvious choices in there:

**Force MJPEG via `CAP_PROP_FOURCC`.** Webcams often default to YUYV (uncompressed), which works fine for one camera but blows past USB 2.0 bandwidth with three. Three uncompressed 1080p30 streams = ~1.8 Gbps; USB 2.0 caps practically around 280 Mbps. With MJPEG (hardware-compressed in the camera), three streams comfortably fit in USB 2 with headroom to spare.

**Disable autofocus.** This was a subtle one. If autofocus is on, the lens position changes mid-session — and that *is* the focal length you're trying to measure. Half your captures end up calibrating one focal length, half another, and the calibration averages them poorly. Locking the focus is essential.

**Save the raw frame, not the annotated preview.** The preview has corner overlays drawn on it; you don't want those baked into the image you'll calibrate from.

**Only allow saving when corners are detected.** This prevents accidentally saving useless frames (board partially out of frame, motion blur). The live "BOARD DETECTED" overlay tells you whether the current pose is usable before you press space.

### The pose variety problem

The capture is the part that requires actual technique. The math wants to fit a model to constrained observations; the more varied the observations, the better-constrained the fit.

What I aimed for across ~25 captures per camera:

- Distance variation: some close (board fills 60-80% of frame), some mid, some far
- Tilt variation: tilts of 30-45° in all four directions
- Frame position: board in each corner of the image and the center
- Rotation: a couple of in-plane rotations

What you want to avoid:

- All captures at the same depth (degenerate for solving focal length)
- All captures with the board parallel to the sensor (degenerate for solving distortion at the edges)
- Motion blur (anytime the board is moving during the shutter)
- Board going off-frame in a corner (corner detection succeeds but the geometry is poorly constrained there)

A good capture set takes about 15-20 minutes of patient board-holding per camera.

### The calibration math

OpenCV does the actual math. The function:

```python
rms, K, dist, rvecs, tvecs = cv2.calibrateCamera(
    objpoints, imgpoints, img_shape, None, None
)
```

What it does internally: for each captured image, the corner detector found 2D pixel locations of the inner corners. We also know the 3D positions of those corners in the board's local frame (we built that grid manually with known square size). The algorithm jointly solves for:

- The intrinsic matrix `K` (one set of values used across all images)
- The distortion coefficients (also shared across all images)
- The 3D pose of the board in each individual image (per-image rotation + translation)

It does this by minimizing the **reprojection error**: for each detected corner, project the known 3D position through the current estimate of camera parameters, get a predicted 2D location, compare to the actual detected 2D location. Total squared error across all corners across all images is the cost function. Levenberg-Marquardt finds the minimum.

The reported RMS error is the square root of the mean squared reprojection error. It's the single most important number out of the calibration:

- **< 0.5 px** is excellent
- **< 1.0 px** is acceptable
- **> 1.5 px** means something went wrong

### My results

```
CAM_A (C920):  RMS = 0.148 px, fx = 913, fy = 918
CAM_B (C930e): RMS = 0.145 px, fx = 761, fy = 765
CAM_C (C930e): RMS = 0.150 px, fx = 756, fy = 761
```

A few satisfying observations from these numbers:

**The two C930e's are nearly identical.** Their `fx` values are within 0.6%, distortion coefficients within 3%. That's not a coincidence — same lens design, same sensor, same manufacturing line. If they'd come back wildly different, that would be a strong signal that something was wrong with the calibration procedure (e.g., I'd somehow mixed up the captures).

**The C920 is meaningfully different.** Its `fx` of 913 is much higher than the C930e's ~760. Higher focal length = narrower field of view. This matches the spec sheet: the C920 has a narrower lens than the C930e. The intrinsics found something real.

**fx ≈ fy in all three.** Within 0.5% of unity ratio. That confirms square pixels (true for all modern sensors) and a reasonable focal length estimate.

**Principal points are within ~20 pixels of image center.** Manufacturing tolerance for sensor placement relative to the lens. Nothing weird.

**Per-image errors are uniform.** Worst single image was 0.036 px on CAM_A. No outliers to drop.

Sub-0.5px RMS calibration on three consumer webcams is a result that researchers publish papers about achieving on industrial cameras a decade ago. Not because it's hard now — OpenCV's calibration is mature and the procedure is well-understood — but as a reminder that "consumer-grade hardware" isn't the bottleneck it once was.

### Saving the results properly

I saved each calibration two ways: a binary `.npz` (efficient for code to load) and a JSON sidecar (human-readable for debugging):

```
calibration/results/calib_CAM_A.npz   # for code
calibration/results/calib_CAM_A.json  # for me
```

The JSON has the same numbers plus metadata: capture date, image count, square size, RMS error. When I look at the calibration files in six months, I want to be able to read them without writing a script.

## Lessons (or, things I'd tell past-me)

**Pin Python to a stable version, not the latest.** Arch ships bleeding-edge; the Python ML ecosystem ships latest-minus-three-months. Use uv (or pyenv) to get a stable Python independent of pacman.

**Always run `pip` as `python -m pip`, or use uv pip.** Bare `pip` resolves to whatever's first in PATH, which on Arch is often the system pip. The system pip operates on the system Python, not your venv. This will eventually bite you.

**Verify your gitignore is actually working before committing.** `git status --ignored` is the diagnostic.

**Label your cameras physically.** Tape and a Sharpie. Webcam indices change; tape doesn't.

**Disable autofocus before calibrating.** Otherwise you're not measuring a single focal length; you're averaging several.

**Force MJPEG for multi-camera USB capture.** USB 2.0 can't carry three uncompressed video streams.

**Measure your checkerboard squares with calipers.** The label lies. The calibration assumes the number you give it is exact.

**A good calibration is sub-0.5 px RMS.** Anything worse, look at per-image errors and drop the bad captures. This usually fixes it.

## What's next

Step 1 establishes each camera's optics. Step 2 establishes their *spatial relationships* — where each camera sits relative to the others, in millimeters and degrees. With both, you can take a 2D pixel from each camera and triangulate to a 3D point in space. With three cameras, you can do this with redundancy (which catches errors and improves accuracy).

The procedure for stereo extrinsics: capture frames from multiple cameras *simultaneously* while the checkerboard is visible to all of them, then run `cv2.stereoCalibrate` with the known intrinsics to solve for the relative pose between camera pairs. With three cameras, you do this for two pairs and chain them, or solve all three jointly.

The interesting design questions for Step 2 are physical, not computational: where do you put the cameras? How much do their fields of view need to overlap? What's the working volume you care about? Those are the next set of decisions.
