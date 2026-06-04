---
layout: post
title: "IK Calibration, the ready() Funeral, and Validating the FK Model"
date: 2026-06-04
categories: [Blueberry Picker]
tags: [phase-2, so-101, kinematics, ik, debugging, calibration]
---

Two sessions, one problem that turned out not to be a problem, and a FK model that ended up more accurate than the tape measure used to check it.

## Where We Left Off

The previous session built `arm_control.py` from scratch: an ikpy kinematic chain over the SO-101 follower arm, with empirically verified joint sign conventions (elbow_flex and wrist_flex both required `sign=-1`, the opposite of what geometry naively suggested), a seeded IK solver that prefers elbow-up configurations, and a `ready()` function that was supposed to sweep the arm from its folded storage position (`SAFE_HOME = [-4.0°, -110.7°, 96.9°, 88.3°, 0.0°]`) to a raised staging position before any pick move.

That session ended with a note in the margin: *ready() jaw collision — investigate further.*

The jaw (gripper tip) was dipping to ~48mm above the shoulder_pan origin during the sweep. Whether that meant it was hitting the desk depended on a coordinate system question nobody had answered yet.

## The Coordinate System Question

The FK model's OriginLink sits at z=0. The shoulder_pan joint is placed at z=SHOULDER_HEIGHT=123.8mm above that. The question was: where is z=0 relative to the desk surface?

The `SHOULDER_HEIGHT` constant was measured as "4 and 7/8 inches from desk to shoulder_lift axis." If the shoulder_lift axis is at z=123.8mm in FK space, then z=0 is 123.8mm below the shoulder_lift axis — which, if the shoulder_lift is 4.875 inches above the desk, puts z=0 at the desk surface.

That means FK z=0 is desk level. And the jaw dipping to 48mm meant 48mm above the desk. That's fine.

But the path safety check added to `move_to()` was using an 80mm threshold on the *elbow* joint height, not the jaw. And the elbow was geometrically passing through ~10mm during the SAFE_HOME → forward target sweep, because the linear interpolation sweeps the shoulder through vertical and the elbow drops with it.

The check was firing and blocking every single move.

## Physical Testing First

Before changing any code, `--manual` mode was used to physically sweep the arm from SAFE_HOME toward a forward position, with FK printing continuously:

```
   pan    lift   elbow   wrist    roll          x        y        z
  -4.0  -111.0   +96.7   +88.2    +0.0      127.8     -8.8     92.7 mm
  -4.0  -107.0   +89.5   +88.2    +0.0      137.0     -9.7     76.5 mm
  -4.0   -67.7   +52.1   +86.1    +0.1      213.9    -15.1     59.7 mm
  -4.0   -64.5   +48.7   +86.1    +0.1      220.4    -15.6     60.0 mm
  ...
  -3.9   +13.1   -36.2   +32.1    +0.1      367.6    -24.9      3.1 mm
  ...
  -2.9   +31.5   -45.8   +30.5    +0.1      379.0    -19.2     75.0 mm
```

The arm was physically safe at every point in this sweep — no desk contact, no crashes. The FK showed z values as low as 3.1mm at one point. The arm didn't touch anything.

The reason: the arm is clamped to the *edge* of the desk. When the shoulder sweeps through vertical and the elbow drops to its geometric minimum, it's passing through a point that is at or off the desk edge — not over the desk surface. The desk doesn't extend under the arm at that angle of travel.

The path safety check was right about the geometry (elbow does dip to ~10mm above the base in FK coordinates) but wrong about the conclusion (it's not hitting anything because there's nothing there to hit). The 80mm threshold was a guess that turned out to be 8× too conservative for this mounting geometry.

The check was removed entirely.

## Killing ready()

While analyzing the manual sweep, a more fundamental question came up: why does `ready()` exist?

The original intent was to provide a staging position — a known-safe configuration from which any IK move could depart without desk interference. The arm would: `home()` → `ready()` → `move_to(target)` → `ready()` → next target.

The previous session had already removed the `ready()` call from inside `move_to()` after it caused a cascade of crashes: motors 2 & 3 drift under load between commands, so a second `ready()` call starting from a drifted-but-not-SAFE_HOME position would sweep through a dangerously different trajectory.

This session revealed the other half of the problem: the arm doesn't need a staging position at all. The natural picking cycle is: `move_to(berry)` → pick → `move_to(basket)` → deposit → repeat. The return-to-basket move brings the arm through a high, clear configuration by necessity. There's no reason to force it through an explicit staging position beforehand.

`READY_POS`, `READY_GRIPPER`, and `ready()` were deleted. The `--test` sequence and all call sites were cleaned up. The only motion primitives that remain are `home()` (fold to storage) and `move_to()` (IK to target).

Simpler is better. If the picking sequence later reveals a need for an explicit waypoint, it can be added with actual evidence for why it's needed and where it should be.

## Link Length Calibration

With the safety check removed and the arm actually moving, it was time to verify the FK model against physical measurement.

The three link lengths in the code were original estimates:

```python
L_UPPER = 0.1143  # shoulder_lift → elbow_flex  (4.5 in)
L_FORE  = 0.1334  # elbow_flex → wrist_flex     (5.25 in)
L_WRIST = 0.1556  # wrist_flex → gripper tip    (6.125 in)
```

Physical measurements (joint axis to joint axis for each):

| Parameter | Old | Measured | Updated |
|---|---|---|---|
| L_UPPER | 4.500" / 114.3mm | 4.500" / 114.3mm | no change |
| L_FORE | 5.250" / 133.4mm | 5.375" / 136.5mm | +3.1mm |
| L_WRIST | 6.125" / 155.6mm | 6.500" / 165.1mm | +9.5mm |

L_WRIST measurement reference point: the rotation axis of the wrist_flex hinge to the tip of the stationary gripper jaw. The moving jaw is not a fixed reference — it shifts with gripper actuation — so it can't be used to anchor a kinematic model.

The lerobot docs don't include link dimensions. The SO-ARM100 repo presumably has CAD, but physical measurement was faster and more trustworthy for this use case.

## Voltage Fix Status

While setting up for testing, `check_servo_voltage.py` was run on motors 2 and 3 — the two servos that had previously required this fix each session because the EEPROM write wasn't persisting.

Both came back clean:

```
Max_Voltage_Limit (reg 14): raw=150  (15.0V)  error_flag=0x00
```

The fix had persisted from a previous session. The startup ritual is no longer required. The script stays in the repo as a diagnostic tool, but it's not a prerequisite for using the arm anymore.

## FK Validation

With the link lengths updated, `--test` was run to validate. The arm moved to all three reachable targets without crashing, dragging, or triggering the gripper on anything. Then `--fk` was run immediately after each move, with the arm holding position, to read back actual joint angles and compare computed FK against the IK target.

**Target 1: (250, 0, 200) — forward, centered**

```
Commanded:  pan=0.0°  lift=+96.4°  elbow=-46.8°  wrist=-74.7°
Actual:     pan=-0.1°  lift=+90.7°  elbow=-39.7°  wrist=-82.8°
FK result:  x=255.0  y=-0.4  z=181.6 mm
Target:     x=250    y=0     z=200 mm
Error:      Δx=5mm   Δy=0.4mm  Δz=18.4mm
```

**Target 2: (200, 80, 200) — forward-left**

```
Actual FK:  x=220.8  y=84.0  z=169.9 mm
Target:     x=200    y=80    z=200 mm
Error:      Δx=21mm  Δy=4mm  Δz=30mm
```

The errors are consistent across targets: x overshoots by 5–20mm, z falls short by 18–30mm, y is accurate. The pattern is a servo compliance signature — the joints don't fully reach their commanded angles under gravity load. Shoulder_lift commanded +96.4° but only reached +90.7°; elbow commanded -46.8° but reached -39.7°. The FK model is correct; the servos are just compliant.

This is expected behavior for hobby-grade bus servos under load without active position correction. It means IK targets should be set 15–25mm higher in z than the true berry position to compensate — or the pick approach should be done slowly enough that the servo has time to settle. Either way, it's a pick sequence tuning problem, not a model problem.

The FK model is validated. Kinematic foundation is solid.

## What's Next

The arm moves where it's told, to within ~20mm. The next step is integrating the berry localizer — the stereo depth pipeline from Phase 1 — with the arm controller, so a detected berry coordinate gets passed to `move_to()` as an actual IK target. That closes the loop from perception to actuation.

Before that: the IK currently produces steep wrist angles (the solution for (250, 0, 200) had wrist at -74.7°) that would make berry grasping awkward. Constraining the IK to prefer wrist-level approach angles is on the list.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — stereo camera rig + YOLO detection + SO-101 arm for autonomous blueberry picking.*
