---
id: tic-bdc2
title: Scaffold the Python project and make the first commit
status: open
type: chore
tier: low
domain: python
epic: minimal-v1
priority: 1
tags:
- scaffold
assignee: null
depends_on: []
references:
- plans/minimal-v1.md
blocked_by: null
created: '2026-11-15T19:46:30'
updated: '2026-11-15T19:46:30'
closed: null
---

## Description
Set up a uv project for a package named acursor, with the layout in plans/minimal-v1.md (adapters, analysis, engine, ui) and a single entry point, python -m acursor. Dependencies: numpy, python-osc, PySide6, dbus-next, PyGObject; add pytest as a dev dependency. The venv needs access to the system gi bindings (GStreamer/PipeWire typelibs), so either use system site-packages or document the alternative. Commit the handoff doc, arbite files and scaffold to main.

Done when: uv run python -m acursor starts and exits cleanly, uv run pytest runs, and main has a first commit.

## Notes
