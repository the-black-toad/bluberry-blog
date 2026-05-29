---
layout: post
title: "Building a Blueberry Picker, Phase 2 Planning"
date: 2026-05-27
categories: [Blueberry Picker]
tags: [planning, phase-2, robotics]
---

# Building a Blueberry Picker, Phase 2 Planning

*May 27, 2026*

Phase 1 is done and Phase 2 is underway — the SO-101 arm kit is in hand, parts are printing, and the first servo IDs are getting configured. Before diving into code, this post captures the planning decisions made for Phase 2 and how they shape Phase 3.

## Where Phase 2 Starts

The Phase 1 output is a list of 3D berry positions: (x, y, z) in mm, ±2mm depth, ±10mm lateral, at ~4fps. Phase 2 takes that list and does something with it — moves an arm to each position, grasps the berry, and pulls it free.

The arm is a SO-101 from TheRobotStudio / HuggingFace lerobot. Six STS3215 serial bus servos, 12V, 30 kg·cm torque. All structural parts 3D printed; servo kit and driver board in hand. Two more servos are on the way before assembly can start.

## Platform: Keep It Simple

The first design question was the platform — what does the arm and camera rig sit on, and should it be designed to eventually bolt onto a rover?

The answer is no. For Phase 2, the platform is a piece of plywood. Arm bolts near the front edge, camera post mounts behind it, cables run back to the laptop. Carry it by hand and set it down wherever it needs to be. No rover-ready hole patterns, no over-engineering for a problem that doesn't exist yet.

The reason is that the picking loop is completely unproven. We don't yet know which failures will dominate, whether the arm reach is sufficient, whether the camera placement is right, or whether the grasp geometry needs rethinking. Building a careful rover-ready platform before any of that is known is a good way to build the wrong thing twice.

## Tethered Power and Data

The original plan had a LiPo battery and Wi-Fi link to the base station. Both are gone for now.

A 12V power cable and a flat outdoor Ethernet cable run from the plywood platform back to the laptop. That's it. Benefits:

- No battery management code to write
- No runtime limit on test sessions
- Lower latency and higher bandwidth than Wi-Fi (important for 4-camera streaming and visual servoing)
- ~$375 less hardware to buy

For a backyard blueberry bush that's ten meters from the house, a tether is not a real constraint. The cable only becomes a problem when the rover needs to navigate autonomously — and the rover isn't happening until the picking loop is reliable.

## Camera Configuration

The 3-camera rig from Phase 1 (three C930e webcams on an aluminum bar, calibrated stereo pairs) mounts on a vertical post ~15-20cm behind the arm base, elevated ~40-50cm, angled ~15° downward. The horizontal bar orientation is preserved — same baselines, same stereo calibration.

This puts the cameras above and behind the arm so the arm doesn't occlude the view to the target berry during approach. The 15° downward tilt angles the FOV into the berry zone rather than straight at branches. Working distance from cameras to berries is ~50-70cm, right in the Phase 1 calibrated range of ~350mm.

The wrist camera (ELP 720P, 100°, ordered) mounts on the end effector and handles the final 5-10cm of approach where the main rig gets occluded by the arm itself.

## Rover: Deferred

The original plan had Phase 3 as "build a rover + get it navigating + integrate the arm + prove picking on the rover." That's two big unknowns stacked on top of each other.

The revised plan: Phase 3 = take the proven Phase 2 picker and put it on wheels. Rover selection and build doesn't happen until Phase 2 produces a reliable pick success rate worth mobilizing.

During the rover research, the LeKiwi mobile base came up — it's the official lerobot mobile platform for the SO-101, with software integration already done. The catch is it uses 3 omni wheels, which work well on hard floors but poorly on dirt and grass. Garden terrain pushed the decision toward a tracked chassis, which means writing the rover control layer from scratch. That's extra Phase 3 work, but it's the right call for the terrain.

The other driver for the deferral: the SO-101 arm can only reach berries in the 10-37cm range above the ground when mounted on a ground-level platform. Highbush blueberry bushes are 1.5-2.7m tall — the arm covers the bottom 25-35% of the canopy. For Phase 2, that's fine; the platform gets hand-carried to wherever the berries are. For Phase 3, it means the rover needs to get within 5-15cm of the bush edge, which puts real precision demands on navigation. Better to understand the picking geometry thoroughly in Phase 2 before committing to a rover design.

## Arm Reach

The SO-101 URDF gives link lengths: upper arm 113mm, forearm 135mm, wrist-to-gripper 61mm. With the shoulder sitting ~10cm above a ground-level platform, the arm sweeps this envelope:

| Arm angle | Forward reach | Height above ground |
|-----------|--------------|---------------------|
| 0° (level) | 31cm | 10cm |
| 30° up | 27cm | 25cm |
| 45° up | 22cm | 32cm |
| 60° up | 15cm | 37cm |

Accessible berry zone: **10-37cm above the ground**, up to **31cm into the bush**. For Phase 2, this is fine — the platform is positioned by hand, close enough to reach the target berries. For Phase 3, the upper canopy remains out of reach without a lift mechanism, which is deferred.

## What's Next

The remaining two servos are on the way. Once they arrive, the servo setup sequence is:

1. `lerobot-setup-motors` to assign IDs 1-6 (lerobot already installed in cv_env)
2. Center each motor to position 2048 before pressing the horn on
3. Assemble per SO-101 assembly guide
4. Mount on plywood platform
5. `lerobot-calibrate` on the assembled arm

After that: inverse kinematics wrapper, hand-eye calibration, and the first attempt at a pick loop.

The ripe season is late June / July. Phase 2 mechanical build can run in parallel with the season arriving.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
