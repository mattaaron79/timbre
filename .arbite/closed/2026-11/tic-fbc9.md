---
id: tic-fbc9
title: Get a SuperCollider server running on Fedora 44 with its own PipeWire stream
status: closed
type: chore
tier: high
domain: audio
epic: minimal-v1
priority: 1
tags:
- supercollider
- pipewire
- spike
assignee: onyx.claude-opus-5.001
depends_on: []
references:
- plans/minimal-v1.md
blocked_by: null
created: '2026-11-15T19:46:30'
updated: '2026-11-15T20:14:22'
closed: '2026-11-15T20:14:22'
---

## Description
SuperCollider is not in the Fedora 44 repos or on Flathub. Find the least painful route: a maintained COPR, or a source build of scsynth and sclang (headless, no IDE, installed under ~/.local). A source build needs devel packages installed with sudo dnf, which Matt has to run: list the exact command for him.

sclang is needed to compile SynthDefs unless a Python SynthDef compiler (for example supriya) is used instead; pick one and say why.

Done when: scsynth boots under pw-jack (or native PipeWire), plays a test sine through the default output, shows up as its own named stream that can be given its own volume, and the smallest glitch-free block/quantum is recorded in a note.

If no route works in reasonable time, stop and report: the fallback is a Python synth, which is Matt's call.

## Notes
- 2026-11-15T20:04:18 onyx.claude-opus-5.001: ROUTE: COPR myriade/supercollider builds supercollider 3.14.1-3.fc44 for fedora-44-x86_64. Installed, no source build needed. Upstream ships no Linux binaries (3.14.1 = macOS/Windows only), Flathub has nothing, Fedora repos retired it (only texlive-sclang-prettifier matches). dnf5-plugin-copr is NOT in the F44 repos, so the repo file was placed by hand:
  curl -sSL https://copr.fedorainfracloud.org/coprs/myriade/supercollider/repo/fedora-44/myriade-superCollider-fedora-44.repo -o /tmp/sc.repo
  sudo cp /tmp/sc.repo /etc/yum.repos.d/ && sudo dnf install -y supercollider
Base package (16 MiB) = scsynth + sclang + supernova + class library, no Qt. The IDE is a separate package and was not installed. No sudo devel packages were needed.

PIPEWIRE: scsynth links pipewire's libjack shim (/usr/lib64/pipewire-0.3/jack/libjack.so.0), so no pw-jack wrapper is needed: `scsynth -d jack` registers node.name=SuperCollider, 2 in / 2 out, 48 kHz, state running. Sample rate 48000, driver block size 1024.

GOTCHA: a bare scsynth has NO SynthDefs. The RPM ships no .scsyndef files and "default" lives in SystemSynthDefs.sc, so /s_new "default" fails with "SynthDef default not found" until something compiles defs. This cost the most time.

sclang HEADLESS: works. `sclang -D file.scd` compiles the class library (~30 s cold, ~1 s cached), boots scsynth, adds a SynthDef and plays. Verified by ear (Matt heard the 440 Hz sine through the default output). Note "Couldn't set realtime scheduling priority 1: Operation not permitted" - harmless here.

OWN STREAM + VOLUME: as a JACK client the node came up media.role=DSP and wpctl said "Node N does not support volume". Fixed most of it with a user-level PipeWire rule: ~/.config/pipewire/jack.conf is a copy of /usr/share/pipewire/jack.conf with one extra jack.rules entry matching application.process.binary = "scsynth" that sets media.role/media.category=Playback, media.title="Acoustic Cursor (scsynth)", node.software-gain=true, node.lock-quantum=false, node.pause-on-idle=false. After systemctl --user restart pipewire the node is role=Playback, auto-connects to the default sink (no pw-link needed) and carries its own title. It still exposes no Volume param: PipeWire 1.6.11 gives jack nodes no software volume even with node.software-gain, and a context.rules drop-in in pipewire.conf.d does not apply to jack nodes at all. So per-stream level must come from the patch's own master gain (tic-6c14 already plans a limiter) or from routing scsynth into a dedicated sink. That is the one Done-when clause not met on the host side.

SMALLEST GLITCH-FREE QUANTUM: 32 (the floor, default.clock.min-quantum=32). Forced-quantum sweep 2048/1024/512/256/128/64/32 with a 16-voice VarSaw+LFNoise2 load, 6 s per step, ERR column from `pw-top -b`: zero xruns at every value, including 32, with no realtime priority. Measured on the Threadripper; re-measure on a slower box before committing to 32.

sclang vs SUPRIYA - PICK: sclang at build time only, no sclang at runtime.
  supriya 26.10b1 installs on Python 3.14 and does compile SynthDefs in Python (built a 206-byte SCgf), but Server.boot() fails against this scsynth with ServerCannotBoot and an empty error_text, and the API is beta (every signature touched differed from the docs: Options has no `audio`, add_parameter is keyword-only, UGens need .ar + `frequency`). Not worth fighting now.
  Chosen shape: author the patch in .scd, compile once with sclang into ~/.local/share/SuperCollider/synthdefs/ - scsynth loads that directory at boot ("Loading synthdefs from default path", seen in its own log) - then drive scsynth from Python over plain OSC. Runtime stays Python + scsynth, one process, no interpreter, and supriya can be revisited when it leaves beta.

REPRO (scratch, this session): quiet.scd / server_up.scd (sclang headless), sweep.sh (quantum sweep), probe.sh (node role/volume), supriya_test.py, osc.py (stdlib OSC sender). pw-record on this host records silence even for audio that is audible, so it is useless as a gate - verify by ear or with a JACK tap.

- 2026-11-15T20:04:19 system: Submitted for review.

- 2026-11-15T20:09:37 claude.fable-5.001: REVIEW (independent re-check, 2026-11-15). Confirmed: supercollider-3.14.1-3.fc44 from COPR myriade/supercollider; scsynth, sclang, supernova in /usr/bin; scsynth links PipeWire's libjack; sclang -D compiles a SynthDef headless; a bare scsynth loaded that def over OSC (/d_loadDir), ran a synth and freed it, at 48 kHz, connected to USB Audio Speakers. The jack.conf rule applies (role=Playback, title 'Acoustic Cursor (scsynth)').

Corrections for tic-6c14:
1. 'scsynth -d jack' is wrong: -d is max-number-of-synth-defs, not a driver. Drop it; JACK is the only backend in this build.
2. A bare scsynth does NOT auto-connect. Without links its node stays suspended, the audio thread never runs, and every OSC command (even /status) times out, which looks like a dead server. sclang hides this by setting SC_JACK_DEFAULT_OUTPUTS. Start scsynth with SC_JACK_DEFAULT_OUTPUTS=system (verified: connects out_1/out_2 to the default sink) or pw-link the ports.
3. Bind with -B 127.0.0.1 and use -i 0 or 2, -o 2; defaults are 8 in, 8 out.

Still open: the node has no Volume param and is not listed under Streams in wpctl, so the 'own volume' clause is unmet, as the worker said. Not re-verified by me: the quantum-32 sweep (driver block size was 1024 in my runs; nothing forced it lower).

Housekeeping outside the repo: ~/.config/pipewire/jack.conf is a full copy of the system file and will shadow package updates; a drop-in under ~/.config/pipewire/jack.conf.d/ holding only the rule would be safer. ~/.config/pipewire/pipewire.conf.d/ was left empty.

- 2026-11-15T20:14:22 claude.fable-5.001: REVIEW FIXES. (1) Own volume: scsynth now plays into a dedicated sink made by pw-loopback (node acoustic_cursor, shown as 'Acoustic Cursor'); wpctl set-volume on it works (1.00 -> 0.50 -> 1.00) and it forwards to the default sink. (2) Config: replaced the full copy of jack.conf with a drop-in, ~/.config/pipewire/jack.conf.d/50-acoustic-cursor.conf, holding only the scsynth rule; removed the empty pipewire.conf.d. The rule still applies (role Playback, title set). (3) Block size: PIPEWIRE_LATENCY=64/48000 and 32/48000 give scsynth driver block sizes 64 and 32, 0 in the pw-top ERR column on a single-synth load. Exact commands and the 'unconnected scsynth looks dead' trap are in plans/minimal-v1.md, section 'Running scsynth'.

Observable: with the pw-loopback and scsynth commands from the plan running, an 'Acoustic Cursor' output device appears in the Plasma volume applet and wpctl status with its own volume; sending /d_loadDir and /s_new over OSC to scsynth plays through it to the speakers.

- 2026-11-15T20:14:22 claude.fable-5.001: Accepted.
