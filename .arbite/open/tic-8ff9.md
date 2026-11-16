---
id: tic-8ff9
title: Motion smoothing and patch analysis in pure numpy
status: open
type: feature
tier: medium
domain: python
epic: minimal-v1
priority: 2
tags:
- analysis
- smoothing
assignee: null
depends_on:
- tic-bdc2
references:
- plans/minimal-v1.md
blocked_by: null
created: '2026-11-15T19:46:31'
updated: '2026-11-15T19:46:32'
closed: null
---

## Description
Implement the handoff doc's 'Motion smoothing and analysis' section with no platform imports.

Smoothing: exponential smoothing of velocity with a speed-dependent time constant, direction frozen below a minimum speed, hysteresis on direction, and a gliding read angle.

Per update, from a luminance patch (default 32 px square): contrast/gain, sharpness threshold with a soft knee, Sobel gradient, grain (variance), mean luminance, the 1D line along the smoothed motion vector, and the perpendicular slice. Return one result object carrying speed, direction, line samples, edge strength, grain and luminance.

Every constant comes from a parameters object so the panel and TOML file can drive it.

Done when: unit tests on synthetic images pass (flat field gives no edge and no grain; a vertical step gives a strong edge when moving horizontally and a weak one moving vertically; noise gives grain; raising the threshold removes a mild gradient but keeps a hard step; jitter around zero speed does not swing direction), and one update on a 32 px patch costs well under 1 ms.

## Notes
