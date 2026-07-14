---
layout: post
title: "Aristotle: An AI Professor With a Gradebook"
date: 2026-07-14
categories: [Meta, Aristotle]
tags: [learning, infrastructure, sre, retrieval-practice, claude-code, tooling]
---

Quick detour from the robot. The other project running on this desk is me — specifically, retooling from VM-era sysadmin work into modern infrastructure engineering. This post is about the system doing the retooling, because it turned out to need the same engineering discipline as the picker.

## The founding failure

On July 8th I got rejected at a recruiter screen. Not the onsite, not the technical phone screen — the *recruiter screen*. The post-mortem was uncomfortable but clear: under pressure I revert to the vocabulary of the infrastructure I grew up on. Boxes instead of workloads. Uptime instead of SLIs. Utilization instead of goodput. The content of my answers was mostly fine; the framing dated me on contact.

Reading more Kubernetes docs wasn't going to fix that. The failure mode was a *habit*, and habits don't respond to input — they respond to reps, grading, and someone refusing to let the relapse slide. So I built the someone.

## Aristotle

Aristotle is a Claude Code persona plus a local web app, and the split of responsibilities matters:

- **The persona** (a `CLAUDE.md` file) is a hard-nosed professor whose charter is explicitly not to be nice. It grades cold, cross-examines answers line by line, and — rule number one — flags every relapse into old-world framing even when the answer is otherwise right.
- **The tracker** (a small Python web app) is the source of record: a skill tree with enforced prerequisites, assignments as JSON, a question bank seeded from real interview post-mortems, and a dashboard of leading indicators.

The dashboard is the part I look at every morning:

![The Aristotle dashboard: countdown to the exam gate, retrieval checks answered, drill streak, and a ledger of unanswered follow-ups](https://pub-7dccfb8281d647ea908c15e9e1c6a9cb.r2.dev/images/aristotle-dashboard.png){: width="972" }

The design principle is the same one that runs the robot project: **artifact or it didn't happen.** Reading a chapter counts for nothing. A skill only moves to "solid" behind a commit, a document, a benchmark, or a cold-mock performance. The "ledger" panel in the middle is literally a list of what I owe the professor — graded answers with unanswered follow-ups don't disappear, they sit there accusing me.

## The rules that make it work

A few mechanisms earned their place the hard way:

**Retrieval checks, not reading notes.** Early on, "graded" meant Aristotle reviewed my notes on a chapter. That's a comprehension theater — notes are written *with the book open*. The grading model now is cold, closed-book retrieval: every assignment carries a set of questions answered from memory, and *those* get the markup. Notes are a private input, never the graded artifact.

**The grading loop terminates.** A markup follow-up gets exactly one response round, then a settled/not-settled disposition. No infinite Socratic spirals. Persistent misses don't get re-argued — they escalate into the drill pool, a spaced re-test, or the exam gate. This one change killed a failure mode where "still discussing follow-up 3" was indistinguishable from progress.

**The prerequisite chain is enforced against me.** The GPU-infrastructure tier — the stuff I actually want to work on — stays locked until Phase 1 exits. I want the thesis topic before passing quals; the professor says no. Having that argument with a config file instead of with my own discipline is, honestly, the point.

**Drills are logged, and a flat week gets called out.** The drill-streak tile on the dashboard is currently showing "today: unlogged" in orange, which means by the time this post is up I'd better have fixed that.

## Why put this on the robot blog

Because it's the same lesson the picker keeps teaching. The desk-slam post was about a signal that pattern-matched a known failure but had a different cause; the fix was simulating before theorizing. The recruiter screen was the same shape: the signal ("I know this material") pattern-matched competence, but the actual failure was elsewhere — in framing, under pressure, unmeasured. The fix, in both cases, was instrumentation and a test harness that doesn't accept my word for it.

Fourteen days to the exam gate. The countdown tile does not care how I feel about it.
