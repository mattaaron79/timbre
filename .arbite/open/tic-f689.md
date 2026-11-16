---
id: tic-f689
title: 'Tuning panel: patch preview, live sliders and a hot-reloading TOML file'
status: open
type: feature
tier: medium
domain: ui
epic: minimal-v1
priority: 4
tags:
- pyside6
- config
- preview
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
A small always-on-top PySide6 window that never takes keyboard focus. It shows the exact filtered patch the analyzer sees (grayscale, threshold applied), overlays the smoothed motion vector, sampled line and perpendicular slice, and has readouts for speed, edge strength and grain.

One slider per mapping value (analysis and sound), generated from the parameter definitions rather than hand-built. All values live in one TOML file: moving a slider writes it, and editing the file in an editor updates the sliders and the sound within a second, without restarting audio. A flag runs the process with no window at all.

Done when: dragging the sharpness slider audibly removes mild variation while keeping hard edges, a hand edit to the TOML file takes effect within a second, and the window does not steal focus when shown or clicked through its sliders with the keyboard elsewhere.

## Notes
