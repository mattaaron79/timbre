---
id: tic-e05e
title: 'Docs: add the QFN delegation section to AGENTS.md and CLAUDE.md'
status: review
type: chore
tier: low
domain: docs
epic: null
priority: null
tags: []
assignee: null
depends_on: []
blocked_by: null
created: '2026-11-15T19:55:55'
updated: '2026-11-15T19:57:59'
closed: null
---

## Description


## Notes
- 2026-11-15T19:56:02 claude.opus-5.001: Added the QFN delegation block (<!-- QFN DELEGATION --> ... END) to AGENTS.md and mirrored it to CLAUDE.md: the two files are byte-identical, 210 lines. Content is carnot's section adapted to this repo: LiteLLM on :4000 (provider litellm, model name picks the machine), pool model is the worker's --model, the Work All loop with health/wait among the actions, the omp fallback, and the qwen.* worker rules. Adapted: branch is main; no .carnot/orchestrate.yaml or suite yet, so the block states the defaults (one pool, the active provider, 3 slots, 'python -m pytest -q') and gives the snippet to add when tests exist; strata is given no slots (unreliable as a worker); carnot-only ticket ids and the CONVENTIONS.md rule dropped. Observable effect: a timbre session or CLI orchestrator can dispatch tickets to local QFN workers and knows the gates, exit codes and its own worker identity scheme. Nothing was committed.

- 2026-11-15T19:56:02 system: Submitted for review.

- 2026-11-15T19:56:19 claude.opus-5.001: Follow-up fix in the same block: the 'harness' bullet claimed workers run from a pinned worktree of the base branch. That harness exists only in the carnot repo (see the orchestrate skill's reference.md: 'harness/ (only when the project is carnot)'). Rewritten for this repo: workers run from the installed carnot. Both files re-written identically.

- 2026-11-15T19:57:59 claude.opus-5.001: Also added CONVENTIONS.md (87 lines) at the repo root, and restored the worker rule 'Read CONVENTIONS.md first, then the ticket and any spec it references.' in both instruction files. Content: keep it simple (this epic is a spike); where code lives (src/acursor/{adapters,analysis,engine,ui}, config.toml, spike/ for throwaways); the platform boundary (only adapters/ imports gi, dbus_next or a portal/KWin API; analysis is pure numpy with no I/O; engine knows OSC only); the sound path (cache at 15-30 fps, analysis at mouse rate, never block it; every mapping a live config.toml parameter; quiet by default with a limiter and an HF ceiling; stillness silent; mono must work); respect the desktop (never take focus or move the pointer, no Orca hotkey clash, own PipeWire stream, load and unload the KWin script); tests (synthetic images for analysis, adapter tests skip without a live session, uv run pytest green on main, listening verdicts recorded in the ticket); STE style; branch <id> and commits prefixed <id>:.
