# Timbre conventions

Read this file before you write code, comments, messages or tickets.
Requirements: `Acoustic Cursor — Handoff & Requirements.md`. Epic shape and machine
facts: `.arbite/plans/minimal-v1.md`.

## 1. Keep it simple

- Use one clear mechanism. Do not make a general one.
- Write code that a person can read from top to bottom.
- Do not add a class, a layer or a setting before a test needs it.
- This epic is a spike. Build the least that answers the ticket, and write what you
  learned in the ticket.

## 2. Where code lives

```
src/acursor/adapters/   portal capture, KWin cursor, global hotkey, D-Bus
src/acursor/analysis/   smoothing, patch, threshold, Sobel, line and slice
src/acursor/engine/     scsynth supervisor, SynthDefs, OSC messages
src/acursor/ui/         preview and sliders
config.toml             every mapping value, watched for changes
```

- Platform code (portal, PipeWire, KWin, D-Bus, AT-SPI) lives only in `adapters/`.
  Nothing outside it imports `gi`, `dbus_next` or a KWin/portal API.
- `analysis/` is pure numpy. It takes arrays and numbers and returns numbers. It does
  no I/O, no threads, no platform import.
- `engine/` knows OSC and SynthDefs. It does not know what a pixel is.
- Spike code goes in `spike/`. Code in `src/` never imports it. Port it with real tests
  and delete the spike in the same commit.

## 3. The sound path

- Capture updates a cached frame at 15-30 fps. Analysis runs at mouse rate from that
  cache. Never block the analysis path on capture, disk or a lock.
- Every mapping value is a live parameter in `config.toml`. Do not hardcode a constant
  that the developer may want to tune. A change must apply without an audio restart.
- Quiet by default: a low master level, a soft limiter, a high-frequency ceiling.
- Stillness is silent. Do not add a sound the ticket did not ask for.
- Mono must work. Pan is an experiment, never a dependency.

## 4. Respect the desktop

- Never take keyboard focus and never move the system pointer. The preview window is
  floating, always-on-top and unfocused.
- Nothing collides with Orca, including its Insert and Caps Lock modifiers. Every
  shortcut is remappable.
- Audio runs on its own named PipeWire stream, so the user can set its volume apart.
- Leave nothing installed behind: load and unload the KWin script from our process.

## 5. Tests

- Test `analysis/` against synthetic images (gradient, grid, noise, text, photo). It
  needs no desktop, no audio and no display.
- An adapter test that needs the live session skips with a reason when the API is
  absent. It must not fail on a headless run.
- Run the suite with `uv run pytest`. Keep it green on `main`, so a red gate is a
  regression.
- Listening is a test the machine cannot run. Record the verdict in the ticket: what
  you heard, with eyes closed, for how long.

## 6. Write in STE style

Use ASD-STE100 (Simplified Technical English) as a style, for tickets, comments,
docstrings, messages and guides.

1. Use short sentences: 20 words or fewer for an instruction, 25 for a description.
2. Write one instruction in each sentence.
3. Use the imperative for instructions: "Run `uv run pytest`".
4. Use the active voice: "`portal_cast` caches the frame", not "the frame is cached".
5. Use simple, common words, and one word for one meaning.
6. Put the condition first: "If the portal token is missing, ask for permission once."
7. Do not build long noun clusters: "the mean luminance of the patch".

### Comments in code

- Write a comment only when the code cannot say it. Say why, not what.
- Put history and discussion in the ticket, and name the ticket ID in the comment.

## 7. Tickets and git

- Every change is a ticket. Claim it first, and narrate with `arbite note`.
- Branch `<id>`, and start each commit message with `<id>:`.
- The ticket is the devlog: record the decision, the alternative you rejected, and the
  measured number (update rate, block size, latency).
- Close with what a person can observe, and how to check it.
