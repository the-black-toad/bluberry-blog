---
layout: post
title: "Building a Blueberry Picker: First Pick Attempts and the Frontal Pivot"
date: 2026-07-07
categories: [Blueberry Picker, Arm Control]
tags: [so-101, visual-servo, hsv, safety, forward-kinematics, debugging]
---

Two days of first real pick attempts. Nothing got picked — but nearly every assumption baked into the pick pipeline got tested against physical reality, and most of them lost. This post covers a safety root-cause that overturned three weeks of theory, a detector swap, a phantom berry, and finally throwing out the entire top-down approach.

## The desk slam was never the sweep

Since mid-June the running theory for "arm slams into the desk on quit" was that `home()`'s single-stage joint-space interpolation sweeps through low-elbow configurations. The planned fix — raise shoulder_lift first, then fold — had been sitting in the todo list.

Before touching the hardware, I simulated it. The FK is pure numpy, so sweeping candidate joint paths and computing the minimum end-effector height along each is cheap. The results were the opposite of the theory:

- The **direct sweep to home never dips**. Across 48 IK-reachable poses and 4,000 random in-bounds poses, the straight interpolation to `[0,0,0,0,0]` never went below its starting height.
- The **planned lift-first fix was actively dangerous** — from some poses it drives the gripper 105mm *below the desk*.

So what was slamming the arm? `SO101Follower.disconnect()`. lerobot's config defaults `disable_torque_on_disconnect: True` — and the shutdown path caught `Exception`, not `BaseException`. A second Ctrl+C during `home()` (a `KeyboardInterrupt`) skipped the rest of homing, fell through to `disconnect()`, torque dropped, and gravity did the slamming. The sweep was innocent all along.

The fix: `home()` now plans — it simulates candidate paths through the FK and executes the first one that keeps the EEF and elbow clear of the desk — and the shutdown path catches `BaseException`, retries `home()` once, and only disconnects after the arm is actually up.

Lesson: a signal that pattern-matches a known failure ("the arm hit the desk, must be the trajectory") can have a completely different cause. Simulate before theorizing.

## The model doesn't like ripe blueberries

With a test berry rigged on a wire loop, YOLO refused to see it. Best confidence: 0.018. A frame capture showed why, twice over: the camera auto-exposed for the bright white wall and rendered the berry as a pure black silhouette — and the training set turns out to be daylight photos of the bush when the berries were mostly *unripe*. Green and white berries on leaves; nothing like a dark ripe berry indoors.

Public pretrained blueberry weights effectively don't exist (plenty of papers, none ship models), so the interim answer was already in the repo: the HSV color-threshold baseline. Wrapped in a ~50-line shim that mimics the tiny slice of the ultralytics results API the pipeline actually touches (`results.boxes`, `.xyxy`, `.conf`), it drops in as `--model hsv` with zero pipeline changes. With a lamp on the berry and a darker background, detection went from 0.02 to stable.

## The phantom berry

First live pick: the arm confidently grasped empty air 130mm to the right of the berry. The log showed it had picked a cluster seen by only *one* stereo pair — a shadow reading as dark blue-purple — while the real berry sat there confirmed by both pairs. Two fixes:

- Clusters must now be seen by **two distinct stereo pairs** to be pickable (counting pairs, not detections — a double-hit in one view doesn't count).
- If the wrist camera sees no berry at the servo point, the pick **aborts** instead of descending blind. That gate would have saved this attempt.

## The pivot: this arm picks from the front

The next attempts kept aborting at the hover, with the wrist camera "staring directly at" the berry but detecting nothing. Saving the servo frame to disk settled it: the wrist cam wasn't looking down at a berry at all. It was looking *horizontally at the wall*, jaws silhouetted in the corner.

The whole pick design assumed a top-down approach: hover 100mm above the berry, look down, descend. But the IK is position-only — the wrist is free, and this arm's natural solution holds the gripper nearly **horizontal**. A horizontal gripper can't look down, and can't descend jaws-first onto anything. Every "miss measurement" I'd been tuning against was actually the arm parked at its commanded hover, never a grasp at all. One bogus transform "correction" got applied and reverted before this clicked.

So the pick sequence got rewritten to match the arm instead of fighting it:

1. **Standoff** 120mm short of the berry along the horizontal ray from the base, at berry height — the wrist cam now points straight at the berry.
2. **Visual servo in full 3D** — the image plane now corrects lateral *and* vertical error (the axes that had been hurting), with depth from stereo. A correction over 80mm aborts as "that's not the berry." A bonus: because the berry is measured relative to the camera and commanded through the same FK, systematic FK bias largely cancels.
3. Open 30%, **advance** onto the berry, close.
4. **Retreat straight back** — pulling the berry off its stem instead of dragging sideways through the rig.

This is also, conveniently, how the robot will eventually have to pick from a real bush: berries hang on the outside face, and you approach them from the front.

## Odds and ends

- `open_camera()` now returns the `/dev/videoN` path instead of an index — the V4L2 backend accepts paths directly, so cameras landing on high device numbers after a replug just work.
- Camera threads are joined before their captures release (an occasional-segfault race).
- `pick_berry.py` finally lives in git. It had been untracked the whole time.

## Where things stand

Detection: stable (HSV stand-in; a YOLO fine-tune on ripe-berry data is the durable fix). Targeting: 2-view confirmed, phantom-proof. Safety: FK-verified homing plus a torque-drop guard. The frontal pick sequence is implemented and compiled but hasn't touched the hardware yet — that's the first thing next session.

Still no berry in the jaw. But for the first time, every remaining unknown is measurable.
