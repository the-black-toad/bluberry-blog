---
layout: post
title: "There and Back Again: A Story of Nuking Your Login with Claude and Fixing It with Claude"
date: 2026-05-29
categories: [Linux]
tags: [linux, dwm, xrdp, debugging, dotfiles]
---

This is not a blueberry post. This is a post about logging into your own computer and getting nothing but a shell for two days, and what it took to trace that back to its source. Claude caused half of it and fixed all of it.

## The Setup

Artix Linux, runit, DWM, LARBS dotfiles. The machine also runs xrdp so I can connect to it remotely — that detail matters. DWM auto-starts from `~/.config/shell/profile` via a guard that checks whether to call `exec startx`:

```zsh
if pacman -Qs libxft-bgra >/dev/null 2>&1; then
    [ "$(tty)" = "/dev/tty1" ] && ! pidof -s Xorg >/dev/null 2>&1 && exec startx "$XINITRC"
```

The logic: if we're on TTY1 and Xorg isn't already running, start X. Simple enough.

## How It Broke

After setting up xrdp, DWM stopped auto-starting on local TTY1 logins. The symptom was clean: log in, get a shell, nothing else. The original guard used `pidof -s Xorg` to check for a running Xorg process. With xrdp active, there's always an Xorg process — it runs one at display `:10` for remote sessions. So `pidof -s Xorg` always returned success, the guard always blocked, and `exec startx` never ran.

First session with Claude: correct diagnosis. The fix was to replace the process check with a lock file check:

```zsh
[ "$(tty)" = "/dev/tty1" ] && [ ! -f /tmp/.X0-lock ] && exec startx "$XINITRC"
```

`/tmp/.X0-lock` is created by Xorg when it claims display `:0` — the local display. xrdp's session at `:10` creates `/tmp/.X10-lock`, not `.X0-lock`. So the new guard correctly distinguishes between a local X session and a remote one.

This fix was written to `~/.config/shell/profile`. It was the right fix. It also didn't work.

## What Was Actually Broken

Second session: DWM still not starting. The problem turned out to be one directory level above the fix — in how zsh was finding its login profile.

The dotfile setup uses ZDOTDIR. `~/.zshenv` sets:

```zsh
export ZDOTDIR="$HOME/.config/zsh"
```

This is a LARBS convention to keep zsh config out of the home directory. Once ZDOTDIR is set, zsh looks for its login profile at `$ZDOTDIR/.zprofile` — that is, `~/.config/zsh/.zprofile`. It does not fall back to `~/.zprofile`.

`~/.config/zsh/.zprofile` existed. It was a symlink:

```
lrwxrwxrwx ... .zprofile -> ../../shell/profile
```

Looks fine. Is not fine. From `~/.config/zsh/`, the path `../../shell/profile` resolves to:

- One `../` up: `~/.config/`
- Two `../` up: `~/`
- Plus `shell/profile`: `~/shell/profile`

`~/shell/profile` does not exist. The symlink was broken. Every time zsh started a login shell, it silently skipped the missing profile, set no environment, ran no `exec startx`, and handed over a plain shell.

The fix was one character:

```bash
ln -sf ../shell/profile ~/.config/zsh/.zprofile
```

One level up instead of two. `../shell/profile` from `~/.config/zsh/` resolves to `~/.config/shell/profile`, which is where the profile actually lives.

## What Each Session Got Right and Wrong

**Session 1** diagnosed the right bug in the right file. The `pidof` guard was the actual problem, and the lock file replacement was the correct solution. What it missed: the file being edited wasn't being sourced by anything. The plumbing between zsh and the profile wasn't verified.

**Session 2** found the broken symlink immediately once it looked for it. `file ~/.config/zsh/.zprofile` reports `broken symbolic link` — not ambiguous. The diagnosis was one command.

The lesson isn't that AI tools are unreliable. It's the standard debugging lesson: a correct fix applied to an unreachable code path is not a fix. Verify the plumbing before verifying the logic.

## The Enter-to-Login Issue

There was a second symptom: on TTY1, the login prompt didn't appear until Enter was pressed. This one is benign. The agetty service for tty1 runs with `--noclear`:

```
# /etc/runit/sv/agetty-tty1/conf
if [ "${tty}" = "tty1" ]; then
    GETTY_ARGS="--noclear"
fi
```

`--noclear` means the terminal isn't wiped before the login prompt appears. Boot messages from the kernel and runit services stay on screen, and the login prompt gets appended below them — sometimes off-screen. Pressing Enter causes agetty to redraw the prompt. Cosmetic issue, not worth changing.

## Current State

DWM auto-starts correctly on TTY1. xrdp sessions at `:10` are unaffected. The lock file check works as intended.

The two-session round trip cost about a day of occasionally-broken local logins, which is fine. The interesting part is that both the cause (the symlink with one extra `../`) and the solution (everything) came out of the same tool. The first session fixed the logic and broke the plumbing. The second session found the plumbing.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
