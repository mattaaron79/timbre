# minimal-v1 — a playable acoustic cursor on this machine

Source of requirements: `Acoustic Cursor — Handoff & Requirements.md` (repo root).
This epic is the spike: just enough to scrub the real desktop with eyes closed and
judge whether the idea is useful. It is not the full v1 from the handoff doc.

## Decisions (Matt, 2026-11-15)

- **Audio engine: SuperCollider**, driven over OSC, as in the handoff doc.
- **Tuning: a slider panel plus a TOML file.** The panel shows the patch preview and a
  slider per mapping; the TOML file hot-reloads on save. Both edit the same values.
- **Scope: bare minimum.** One process started from a terminal, with a global on/off
  hotkey.

## Out of this epic

systemd service, daemon/client split, `acursor` CLI, presets, shake-to-ping, radius
modes other than the cursor square, hold mode, Orca ducking, latency under 20 ms as a
hard gate (measure it, don't block on it).

Keep the handoff doc's seams anyway: platform code (portal, KWin, D-Bus) stays in an
adapter package, and analysis stays pure numpy with no platform imports.

## Machine facts (checked 2026-11-15)

- Fedora 44, KDE Plasma 6.8.2, KWin 6.8.2, Wayland session.
- PipeWire 1.6.11 with `pipewire-jack-audio-connection-kit` and `pipewire-gstreamer`;
  xdg-desktop-portal 1.22.1 with the KDE backend 6.8.2.
- Three monitors, all scale 1, in one global coordinate space:
  - DP-5 3440x1440 at (0, 265)
  - DP-9 1200x1920 at (3440, 0) — portrait
  - VGA-1 1920x1200 at (5360, 0)
- Python 3.14 with `uv`. System site-packages already has numpy, PySide6, PyGObject
  (`gi`), dbus-python and dbus-next. Missing: python-osc, OpenCV, scipy, watchdog.
- Orca is installed.
- SuperCollider 3.14.1 is installed from the COPR `myriade/supercollider` (it is not in
  the Fedora 44 repos). `scsynth`, `sclang` and `supernova` are in `/usr/bin`; no IDE.

## Running scsynth (settled in tic-fbc9)

- SynthDefs are written in `.scd` and compiled with headless `sclang -D`; at run time
  Python talks plain OSC to `scsynth`, with no sclang process.
- `scsynth` is a JACK client of PipeWire directly; no `pw-jack` wrapper, no `-d` flag.
- It plays into its own sink, which is what gives the cursor its own volume:

      pw-loopback -n acoustic_cursor \
        --capture-props='media.class=Audio/Sink node.name=acoustic_cursor node.description="Acoustic Cursor"' \
        --playback-props='node.name=acoustic_cursor.out node.description="Acoustic Cursor" media.role=Accessibility'

      PIPEWIRE_LATENCY=64/48000 SC_JACK_DEFAULT_OUTPUTS="Acoustic Cursor" SC_JACK_DEFAULT_INPUTS="" \
        scsynth -u <port> -B 127.0.0.1 -i 0 -o 2

  The process should own both: start the loopback first, stop it on exit.
- **An unconnected scsynth looks dead.** With no links its node is suspended, the audio
  thread never runs, and every OSC command times out, `/status` included. A bare
  `scsynth` connects nothing unless `SC_JACK_DEFAULT_OUTPUTS` is set.
- `PIPEWIRE_LATENCY` sets the block size per client: 64 and 32 both ran with no xruns
  on a light load. Start at 64 (1.3 ms).
- `~/.config/pipewire/jack.conf.d/50-acoustic-cursor.conf` (outside the repo) marks the
  scsynth node as a playback stream titled "Acoustic Cursor (scsynth)".

## Shape

```
adapters/kwin_cursor   KWin script -> D-Bus -> (x, y) in global coordinates
adapters/portal_cast   ScreenCast portal -> PipeWire -> cached luminance frame(s)
adapters/hotkey        KDE global shortcut -> toggle
analysis/              smoothing, patch, threshold, Sobel, line + perpendicular slice
engine/                scsynth supervisor, SynthDefs, OSC messages
ui/                    preview + sliders (never takes focus)
config.toml            every mapping value, watched for changes
```

Capture updates the cached frame at 15–30 fps; analysis runs at mouse rate from the
cache and sends OSC.

## Order

The three unknowns (SuperCollider install, cursor position, capture) are independent
and go first. Analysis is pure and can be built against synthetic images in parallel.
Everything else waits for the first end-to-end sound.
