---
layout: post
title: "SO-101 Arm Setup: Drivers, Dead EEPROM Defaults, and the 8V Problem"
date: 2026-06-02
categories: [Blueberry Picker]
tags: [phase-2, so-101, lerobot, servos, debugging, linux]
---

The SO-101 arm is assembled. Getting there took most of a day, and the interesting part had nothing to do with the arm — it was a chain of three separate problems that each looked like the real problem until the next one showed up underneath it. This post is a forensics writeup.

## The Setup

Six STS3215 servos, 12V variant, Waveshare Bus Servo Adapter (A) controller board, CH343P USB-to-serial chip. lerobot is installed in `cv_env`, `lerobot-setup-motors` is the first step. It scans for each motor one at a time, assigns IDs 1–6, and sets the baud rate. Straightforward.

Nothing about it was straightforward.

## Problem 1: Wrong Driver

The CH343P enumerates as `idVendor=1a86, idProduct=55d3`. On boot, the kernel loads `ch341` — a driver for a different WCH chip — and binds it to this device. The ch341 driver creates `/dev/ttyUSB0` and `/dev/ttyUSB1`, which look correct but fail with `ENODEV` on any write.

The fix is to load the WCH `ch343ser_linux` driver from source:

```bash
cd /tmp
git clone https://github.com/WCHSoftGroup/ch343ser_linux.git
cd ch343ser_linux/driver
make
sudo insmod ch343.ko
```

Once ch341 is unloaded (`sudo modprobe -r ch341`) and the board is replugged, ch343 claims it correctly and creates `/dev/ttyCH343USB0`.

That fixes the port. Next problem.

## Problem 2: The Udev Rule That Was Causing Problem 1

After writing a udev rule to automate this on future boots, I found one already existed at `/etc/udev/rules.d/99-ch341-55d3.rules`:

```
SUBSYSTEM=="usb", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="55d3", RUN+="/sbin/modprobe ch341"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="55d3", RUN+="/bin/sh -c 'echo 1a86 55d3 > /sys/bus/usb-serial/drivers/ch341-uart/new_id'"
```

`1a86:55d3` is not in ch341's default USB ID table. ch341 only claims this device because someone previously wrote a rule to force it. The rule was the cause of Problem 1.

It was replaced with:

```
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="55d3", RUN+="/sbin/modprobe ch343"
```

The ch343 module was installed permanently via `sudo cp ch343.ko /lib/modules/$(uname -r)/kernel/drivers/usb/serial/ && sudo depmod -a`. After a reload (`udevadm control --reload-rules`), the driver situation is correct on every boot.

## Problem 3: The Actual Problem

With the port working, `lerobot-setup-motors` got further — it found the motor, logged a voltage error, and crashed:

```
ERROR:lerobot.motors.feetech.feetech:Some motors found returned an error status:
{1: '[RxPacketError] Input voltage error!'}
...
termios.error: (22, 'Invalid argument')
```

Two errors, two causes. The termios crash comes second but it's a symptom: when a motor responds with an error status, lerobot's `_read_model_number` skips it, `broadcast_ping` returns an empty dict, and the baud rate scan falls through to 250,000 bps — a non-standard rate that the ch343 driver doesn't support via termios. The termios crash is lerobot going down a dead code path because the real problem got in the way earlier.

The real problem is the voltage error.

### Diagnosing the Voltage Error

The supply is 12V. The board power LED is on. The servo is communicating — it's responding to pings, just with an error flag set. So the voltage is reaching the servo. The question is what threshold it's being compared against.

The STS3215 has `Min_Voltage_Limit` and `Max_Voltage_Limit` registers in EEPROM. A quick diagnostic script using scservo_sdk directly (bypassing lerobot's error-checking) read them back:

```
Max_Voltage_Limit (reg 14): raw=80  (8.0V)  error_flag=0x01
Min_Voltage_Limit (reg 15): raw=40  (4.0V)  error_flag=0x01
Present_Voltage   (reg 62): raw=123 (12.3V) error_flag=0x01
```

The servo is reading 12.3V. Its max voltage limit is set to 8.0V. These are 7.4V motor defaults shipped in 12V motor hardware.

At 12.3V input on a servo with an 8.0V ceiling, the error flag is set on every response, `_read_model_number` skips the motor, and lerobot never gets past the scan.

### The Fix

Writing to the EEPROM required two things that aren't obvious:

1. **Use `write1ByteTxOnly` instead of `write1ByteTxRx`.** When a motor is in error state, it won't send a status packet in response to write commands. `write1ByteTxRx` waits for a response that never comes and reports failure (or a false success). `write1ByteTxOnly` fires and forgets — correct for a servo with Return_Level=0.

2. **Clear the Lock register first.** The STS3215 has a `Lock` register (reg 55) that write-protects EEPROM. Writing the voltage limit without clearing it first does nothing silently.

The fix sequence:

```python
pkt.write1ByteTxOnly(ph, MOTOR_ID, REG_TORQUE_ENABLE, 0)  # disable torque
pkt.write1ByteTxOnly(ph, MOTOR_ID, REG_LOCK, 0)           # unlock EEPROM
pkt.write1ByteTxOnly(ph, MOTOR_ID, REG_MAX_VOLTAGE, 150)  # set max to 15.0V
pkt.write1ByteTxOnly(ph, MOTOR_ID, REG_LOCK, 1)           # lock EEPROM
```

After a power cycle to clear the error flag, the servo reads back clean:

```
Max_Voltage_Limit (reg 14): raw=150 (15.0V) error_flag=0x00
Present_Voltage   (reg 62): raw=123 (12.3V) error_flag=0x00
```

All six motors had the same wrong defaults. The fix was applied to each one before running `lerobot-setup-motors`, which then completed without issues.

The scripts (`check_servo_voltage.py` for diagnosis and fix, `center_servos.py` for pre-assembly centering) are in `arm/` in the project repo.

## Assembly

The SO-101 structural parts are 3D printed. Some joints needed significant sanding before the servo bodies would seat. All faceplates on this build are round — no positional horns — so centering before attachment wasn't needed. The bottom horn press-fits onto the servo's idler nub; the fit was loose enough to require CA glue to stay seated.

The arm is assembled. Calibration is next.

## Summary

Three problems in series:
1. Wrong driver (`ch341`) — fix: build and install `ch343ser_linux`, replace the udev rule that was causing it
2. Voltage error from wrong EEPROM defaults (8.0V max on 12V motors) — fix: clear Lock register, write correct limits via `write1ByteTxOnly`
3. termios crash on 250kbps — this disappears once Problem 2 is fixed; it was never the real issue

The voltage defaults bug will affect anyone who buys 12V STS3215 motors through certain suppliers. The symptom is `Input voltage error` immediately on any ping even with a correct 12V supply connected.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
