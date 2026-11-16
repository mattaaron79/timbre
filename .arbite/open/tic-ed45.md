---
id: tic-ed45
title: Duck cursor audio while Orca speaks, and audit hotkeys against Orca
status: open
type: feature
tier: high
domain: audio
epic: screen-reader-coexistence
priority: 9
tags:
- orca
- ducking
- hotkey
assignee: null
depends_on: []
blocked_by: null
created: '2026-11-15T19:46:32'
updated: '2026-11-15T19:46:32'
closed: null
---

## Description
Tracking ticket required by the handoff doc ('Important: coexist with a screen reader'). Lower or pause the cursor's sound while the screen reader is speaking, then resume; speech always wins. Audit all default shortcuts against Orca's, including Insert and Caps Lock combinations. Not part of minimal-v1, but that epic must keep the cursor on its own PipeWire stream and its shortcuts remappable so this can land later.

## Notes
