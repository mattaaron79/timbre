---
id: tic-71f8
title: 'Spike: global cursor position from a KWin script over D-Bus'
status: open
type: feature
tier: high
domain: plasma
epic: minimal-v1
priority: 1
tags:
- kwin
- dbus
- adapter
- spike
assignee: null
depends_on:
- tic-bdc2
references:
- plans/minimal-v1.md
blocked_by: null
created: '2026-11-15T19:46:30'
updated: '2026-11-15T19:46:32'
closed: null
---

## Description
Confirm on KWin 6.8.2 that a KWin script can report the global cursor position to our process over D-Bus (workspace.cursorPos and its change signal, calling a method on a service we own). Load and unload the script from our process at start and stop, so nothing is left installed in KWin when we exit. No evdev.

Put it behind an adapter that yields (x, y, timestamp) in global coordinates.

Done when: a small test program prints positions while the mouse moves across all three monitors, coordinates match the layout in the plan (0..7280 wide), the update rate is measured and noted, and the pointer and keyboard focus are never touched. If the script route cannot reach a useful rate, note what was tried and the next best option.

## Notes
