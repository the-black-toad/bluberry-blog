---
layout: post
title: "Rebuilding the Dataset with the Webcam"
date: 2026-05-31
categories: [Blueberry Picker, Machine Learning]
tags: [yolo, roboflow, dataset, webcam, opencv]
---

The DSLR dataset that got the project through Phase 1 detection has a fundamental problem: the inference pipeline runs on C920 webcam frames, not DSLR images. Different sensor, different color response, different sharpness characteristics. The model trained on DSLR images works, but it's working despite the domain gap, not because the training data matches reality. Time to fix that.

## The Domain Gap Problem

When I trained YOLOv8n on the 116 DSLR images, I noted in the task log that the plan was to "supplement with webcam-captured images when ripe season arrives." That was kicking the can. The right call is to redo the dataset entirely with webcam captures so the training distribution matches what the model sees at inference time — same sensor, same resolution (1024×576), same lens distortion, same MJPEG compression artifacts.

## One Camera, Not Three

The rig has three cameras. The natural question is whether to capture from all three simultaneously for angle diversity. The answer is no, for a practical reason: three simultaneous angles means three times the labeling work for images that are geometrically similar. The cameras are 4–8 inches apart on a horizontal bar — that's not enough separation to produce meaningfully different perspectives on a berry cluster.

Instead: capture from CAM_A (the center C920) and get the diversity through deliberate framing. Different distances, different angles (looking up into the canopy vs. straight on), different lighting, different cluster densities. That kind of diversity improves generalization more than a few inches of baseline separation.

## Single Class

The previous dataset had a `blueberry` class with plans to add `ripe` and `unripe` subclasses. The cleaner architecture is a single `berry` detector feeding into a separate lightweight ripeness classifier — each model does one job, and the detector doesn't need to be retrained when the ripeness logic changes. So the new dataset uses a single `berry` class throughout.

## Portability: USB Path vs. Model Name

The existing camera infrastructure in `cameras.py` identifies cameras by their stable USB path (`usb-0000:00:14.0-9.3`). That's the right approach for the three-camera stereo rig, where you need to know which physical camera is which. For dataset capture it's the wrong approach — the goal is to unplug from the powered hub, walk to a bush, and capture, which means the camera lands on whatever USB port is convenient.

`dataset/capture_dataset.py` sidesteps this by scanning `v4l2-ctl --list-devices` for the C920 by model name:

```python
def find_c920():
    out = subprocess.check_output(["v4l2-ctl", "--list-devices"],
                                  stderr=subprocess.DEVNULL).decode()
    for block in out.strip().split("\n\n"):
        if "C920" in block:
            match = re.search(r"/dev/video(\d+)", block)
            if match:
                return int(match.group(1))
    return None
```

Plug it in anywhere, run the script, it finds the camera. Images save to `dataset/raw/` with four-digit sequential filenames so sessions accumulate without overwriting.

Capture is triggered by spacebar or the right arrow key — the right arrow lets you hold the camera steady with one hand and trigger with the other on the laptop keyboard.

## What's in the Dataset So Far

61 images captured today of a single unripe bush in midday sun. A few observations from reviewing them:

**The berries aren't ripe.** It's the end of May — everything is white, green, and pink. That's fine. These go into the dataset as-is and teach the model what a berry looks like regardless of color. The plan is ~150 unripe images now, then match it with ~150 ripe images at harvest in a few weeks.

**Close-up shots are blurry.** The C920's autofocus doesn't lock reliably at 8–16 inches. The script already disables autofocus (`CAP_PROP_AUTOFOCUS, 0`), but that locks at whatever distance the camera happened to be focused at when it opened. The practical fix is to point the camera at something at picking distance before triggering capture, let it focus, then start shooting. The blurry close-ups in today's session will be culled before labeling.

**Direct sun overexposes.** Several frames have blown highlights that wash out berry detail. Overcast light or open shade will produce better training images — something to keep in mind for future sessions.

## The Labeling Plan

Same workflow as before: upload to Roboflow, use Smart Select to auto-annotate, review and clean up, generate a versioned dataset with augmentations, export YOLOv8 format. The existing trained model can be used for Roboflow's auto-label feature, which should speed up annotation significantly compared to the first round.

The target before starting labeling is 150+ images. At 61 today, a few more sessions across different bushes and lighting conditions will get there.

## What's Next

- Keep capturing: more bushes, overcast days, different distances and angles
- Cull the blurry/overexposed frames before uploading to Roboflow
- Label and train once the unripe set hits ~150 images
- Return at harvest for the ripe captures and retrain

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
