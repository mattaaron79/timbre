---
id: tic-6c14
title: 'Sound engine: needle texture, carrier, edge blips and limiter in SuperCollider'
status: open
type: feature
tier: high
domain: audio
epic: minimal-v1
priority: 2
tags:
- supercollider
- synthdef
- osc
assignee: null
depends_on:
- tic-bdc2
- tic-fbc9
references:
- plans/minimal-v1.md
blocked_by: null
created: '2026-11-15T19:46:31'
updated: '2026-11-15T20:14:22'
closed: null
---

## Description
The process starts and supervises scsynth, loads the SynthDefs, and exposes a small Python API that turns analysis results into OSC.

Voices, per the handoff doc's 'Sound design v1': a needle texture voice reading the line samples from a buffer, with speed setting playback rate and density; a very quiet carrier bed that is present only while moving; soft, short, filtered edge blips whose level follows edge strength; perpendicular slice shaping timbre (filter brightness), not pan. Mono. Master chain with a quiet default level, soft limiter and high-frequency ceiling. Short distinct earcons for on and off.

All mapping values are controls set over OSC so they can change while sounding. Stopped means silent: gate everything on speed. SynthDef source lives in the repo and can be reloaded without restarting the process.

Done when: a script that feeds synthetic analysis results (a sweep over a step edge, a noisy patch, a flat patch, speed ramping to zero) produces clearly different sounds for each, falls silent at zero speed, and never exceeds the limiter.

## Notes
- 2026-11-15T20:14:22 claude.fable-5.001: Read plans/minimal-v1.md section 'Running scsynth' and the review notes on tic-fbc9 before starting: launch flags, the dedicated pw-loopback sink, and why an unconnected scsynth times out on every OSC command.
