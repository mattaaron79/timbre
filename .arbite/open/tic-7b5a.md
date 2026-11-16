---
id: tic-7b5a
title: 'Wire cursor, capture and analysis to the engine: first end-to-end sound'
status: open
type: feature
tier: high
domain: python
epic: minimal-v1
priority: 3
tags:
- integration
assignee: null
depends_on:
- tic-71f8
- tic-4668
- tic-8ff9
- tic-6c14
references:
- plans/minimal-v1.md
blocked_by: null
created: '2026-11-15T19:46:31'
updated: '2026-11-15T19:46:31'
closed: null
---

## Description
Join the pieces in the one process: on each cursor update, take the patch from the cached frame, run analysis, send OSC. Capture keeps refreshing the cache at its own rate. Cap the update rate at about 250 Hz. Shut everything down cleanly on Ctrl+C (scsynth, portal session, KWin script).

Done when: running python -m acursor and moving the mouse over the real desktop gives texture, carrier and edge blips; holding still is silent; scrubbing left and right over a window border gives a blip per crossing; it works on all three monitors. Record a rough mouse-to-sound latency figure in a note (target under 20 ms, not a gate for this epic).

## Notes
