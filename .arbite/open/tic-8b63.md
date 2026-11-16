---
id: tic-8b63
title: Global hotkey toggles the cursor on and off, with earcons
status: open
type: feature
tier: medium
domain: plasma
epic: minimal-v1
priority: 4
tags:
- hotkey
- kglobalaccel
- adapter
assignee: null
depends_on:
- tic-7b5a
references:
- plans/minimal-v1.md
blocked_by: null
created: '2026-11-15T19:46:31'
updated: '2026-11-15T19:46:31'
closed: null
---

## Description
Register one global shortcut through the KDE global shortcuts API (KGlobalAccel over D-Bus, or the GlobalShortcuts portal) that toggles the cane whatever app has focus. Off stops capture and silences the engine; the process keeps running. On and off each play their earcon.

The default binding must not clash with Orca's shortcuts, including its Insert and Caps Lock modifier combinations, or with existing Plasma bindings; it must be remappable in System Settings. Toggle mode only; hold mode is out of this epic.

Done when: the hotkey works with a different app focused, CPU use while off is near zero, and the chosen default is noted with the check against Orca's bindings.

## Notes
