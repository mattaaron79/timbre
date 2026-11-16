# Acoustic Cursor — Handoff & Requirements

Oct 12, 2026 · @Matthew Aaron

## Overview

Build a prototype of an audio-only "acoustic cursor" that lets a blind or low-vision user feel what is under the mouse on a Linux desktop by sound. It is a personal, open software project, not a commercial product. The first milestone is a sound lab running live on the developer's own Fedora / KDE Plasma screens.

The tool works at the system level, not inside any one application. A pixel-based fallback (screen capture, later OCR and a small vision model) works everywhere; real text and metadata from the accessibility bus come later.

There are three fronts, built in this order:

1. **Texture and edge feel (v1 focus).** Moving the mouse produces a quiet, tactile sound read from the pixels under the cursor, like a record needle on a groove. Edges register as soft blips; grain registers as grit.
2. **Positioning.** Knowing roughly where on screen you are, without relying on stereo. Ideas only for now (see Open explorations).
3. **Descriptions.** Fast, relevant spoken or audio summaries of a region (text, images, UI). Deferred.

Guiding principles:

- **Movement is the primary input.** Sound is driven by the mouse's motion vector; the direction of travel defines the main axis of what is heard.
- **Quiet and non-fatiguing.** Nothing loud or jarring; stillness is silent.
- **Like a cane.** Flipped out when needed, put away otherwise.
- **Tunable everything.** Every mapping is a live parameter, because the right sound is unknown and will be found by experiment.
- **Don't assume sight.** Mappings that feel intuitive while looking can fall apart with eyes closed. Test without the screen.

A related earlier project, Timbre, explored intuitive "vibe" sounds from properties like color and patchiness; its lessons may feed the texture voice later.

## Important: coexist with a screen reader

The acoustic cursor must work alongside a screen reader, not replace it. Most blind users will keep Orca running for text, focus and keyboard navigation, so this tool is an addition to their setup.

Requirements:

- **Audio ducking.** Lower or pause the cursor's sound while the screen reader is speaking, then resume. Speech always wins.
- **No hotkey collisions.** Default shortcuts must not clash with Orca's (including its Insert / Caps Lock modifier combinations). All shortcuts are remappable.
- **No focus stealing.** The daemon and the preview window never take keyboard focus or move the system pointer.
- **No duplicate announcements.** When descriptions arrive later, they must not repeat what Orca already says for the focused element.
- **Separate audio stream.** The cursor's audio runs on its own PipeWire stream so the user can set its volume independently.

Ducking and hotkey rules can land after the first sound-lab milestone, but the architecture must leave room for them from the start. A ticket should track this.

## Scope

v1 is a live sound lab for texture and edge feel on the developer's real screens, grayscale only.

| Area | In v1 | Later |
| --- | --- | --- |
| Input | Live screen capture + global cursor position on Plasma Wayland | X11 adapter, AT-SPI text and metadata |
| Image | Grayscale luminance only | Color and Timbre-style "vibe" voices |
| Sound | Needle/Doppler texture voice, quiet carrier, edge response, sharpness threshold | Richer timbres, orientation cues, grid lines |
| Positioning | None required; stereo not relied on | Quadtree or grid cues, percentile pitch ticks |
| Ping | Shake gesture detected, calls a stub | Real locate/describe behaviour |
| Descriptions | None | OCR, small vision model, spoken summaries |
| Screen reader | Separate audio stream, remappable hotkeys | Ducking while Orca speaks |
| Tooling | Preview window, live config, presets, CLI | Recording and A/B test harness |

Out of scope for now: packaging for others, Windows or macOS, and any commercial concerns.

## Interaction model

The user flips the cursor on like a cane, moves the mouse to feel the screen, and hears nothing when still.

**On / off (the cane)**

- **Toggle mode:** a global hotkey turns it on; the same hotkey turns it off.
- **Hold mode:** active only while a modifier key is held.
- Both modes are supported; which is default is a config choice.
- Each state change plays a short, distinct earcon (on, off), so the user always knows whether it is listening.

**Radius (what the cursor "sees")**

- Variable from the start. Default: a **32 px** square around the cursor.
- Modes: cursor rectangle (adjustable size), focused pane or window, whole screen.
- Radius changes are hotkey-driven and confirmed with an earcon.
- In v1, pane and whole-screen modes only change the analysis region; what they should sound like is open.

**Motion**

- Sound is produced by movement. The smoothed direction of travel is the main axis of sound; the perpendicular axis shapes timbre.
- Moving horizontally reads across a row, so vertical edges register; moving vertically reads down a column, so horizontal edges register. No separate mode is needed.

**Stillness**

- Stillness is silent. No dwell sounds or automatic summaries.
- Sound at rest only comes from an explicit command.

**Shake gesture → ping**

- Detect a shake as several rapid direction reversals in a short window (starting point: 3 or more within 500 ms, tunable).
- A shake calls `ping()`, which is a stub in v1. Its eventual job (locate, describe region, orientation) is undecided.
- Inspired by Plasma's shake-to-enlarge cursor feature.

## Sound design v1

The first stab is a needle reading the pixel field as it streams past the cursor, with mouse speed acting like a Doppler shift. Treat every mapping below as a starting point to tune, not a decision.

**Texture voice (the needle)**

- Sample luminance along the smoothed motion vector through the cursor patch and treat that 1D line as a waveform or grain source.
- Mouse speed sets playback rate and density: fast movement is higher and denser, slow is a low rumble, stopped is silent.
- Grainy or noisy regions should come through as grit; flat regions as near-silence.
- Implementation candidates in SuperCollider: wavetable read (BufRd / Osc on a buffer refilled each frame) or granular playback (GrainBuf), driven by speed.

**Edge response**

- Strong luminance steps along the path produce soft, short blips: tactile, not clicky.
- Edge strength sets blip level; a filtered envelope keeps them gentle.
- Optional later: a distinct timbre for edges perpendicular vs. parallel to motion, and duller, downward-toned blips for large "environment" edges such as window borders.

**Carrier**

- A very quiet, steady bed under the texture, like the faint surface noise of a record, so movement always has a floor of sound.
- Must be clearly separable from texture and never grating. Level and color are tunable.

**Perpendicular axis**

- The slice of the patch perpendicular to motion shapes timbre (for example filter brightness or spectral width), not pan.

**Sharpness threshold**

- A gradient threshold with a soft knee, like a levels or sharpness filter. At low settings the needle ignores mild variation and only catches on hard edges.
- Paired with an analysis-side contrast/gain control.
- Doubles as the main fatigue and clarity control.

**Stereo and position**

- Do not rely on stereo in v1. Mono must work.
- Pan, if used at all, is an optional experiment, not a requirement.

**Safety and comfort**

- Quiet master level by default, a soft limiter, and a high-frequency ceiling.
- All of these are live parameters.

## Motion smoothing and analysis

The motion vector must be smoothed so micro-jitter never swings the reading direction wildly.

**Smoothing**

- Exponential smoothing on velocity, with a speed-dependent time constant: longer at low speed (where jitter dominates), shorter at high speed.
- Freeze direction below a minimum speed instead of letting it wander.
- Hysteresis on direction so small wobbles don't flip the axis.
- Direction changes glide the read angle rather than snapping it.
- All constants are live parameters.

**Per-update analysis**

1. Take the latest grayscale luminance frame (cached from capture).
2. Extract the patch inside the current radius around the cursor.
3. Apply contrast/gain and the sharpness threshold.
4. Compute local gradient (Sobel gx, gy), variance (grain) and mean luminance.
5. Sample the 1D line along the smoothed motion vector, and the perpendicular slice.
6. Send speed, direction, line samples, edge strength, grain and luminance to the audio engine.

**Shake detection**

- Count direction reversals above a speed threshold within a sliding window; fire `ping()` past the count threshold, with a cooldown so one shake fires once.

## Architecture

A long-running Python daemon owns capture, analysis and control; SuperCollider makes the sound; thin clients only talk to the daemon.

&#91;embedded content: system architecture · daemon, adapters, engine, clients\]

Platform-specific parts (portal, KWin, D-Bus, AT-SPI) live only in the adapter layer, so the rest of the code never touches Plasma directly.

- **Control interface:** D-Bus or a local socket. Every client (CLI, preview window, hotkey handler) uses the same commands.
- **Daemon to engine:** OSC to SuperCollider's server, which the daemon starts and supervises.
- **Update rates:** synth parameters update at mouse rate from the cached frame; capture runs separately at 15 to 30 fps.
- **Lifecycle:** cane off stops capture and silences the engine; the daemon itself keeps running.

## Platform: Fedora / KDE Plasma (Wayland)

Target the developer's Fedora Plasma Wayland session from day one, and hide its plumbing behind adapters so the rest of the code never sees it. The developer wants this layer to "just work."

- **Screen capture:** xdg-desktop-portal ScreenCast via PipeWire. Ask for permission once and persist it with the portal's restore token so later sessions reconnect silently. Multi-monitor must work.
- **Cursor position:** captured frames often omit the cursor or its exact position, so track it separately. Preferred: a small KWin script that reports the global cursor position over D-Bus. Avoid reading raw evdev devices, which needs extra permissions.
- **Global shortcuts:** the KDE / portal global shortcuts API, so the hotkeys work whatever app has focus.
- **Audio:** PipeWire (SuperCollider via its JACK compatibility layer, or plain PipeWire output), on its own named stream.
- **Startup:** a systemd user service that starts at login.
- **Later:** AT-SPI adapter for real text and metadata; optional X11 adapter for portability.

Risk to confirm early: the exact KWin and portal APIs on the installed Plasma version. Spike this first, since it shapes the whole input layer.

## Performance targets

Edge feel only reads as tactile if sound follows the mouse almost instantly, so latency is the main constraint.

| Measure | Target |
| --- | --- |
| Mouse movement to audible change | Under 20 ms |
| Cursor position and analysis rate | Mouse poll rate (cap at about 250 Hz) |
| Screen capture rate | 15 to 30 fps, cached as a luminance buffer |
| Audio buffer | Small enough to meet the 20 ms target without glitches |
| CPU when the cane is off | Near zero: capture stopped, engine silent |

Decouple the two rates: capture updates the cached frame slowly, while the cursor path is sampled from that cache at mouse rate. The screen rarely changes as fast as the mouse moves.

## Developer tooling

The project lives or dies on fast experiment loops, so tuning a sound must take seconds, not a rebuild.

**Preview window**

- Small, floating, always-on-top, optional. Never takes focus.
- Shows the exact filtered patch the analyzer sees: grayscale, radius mask, sharpness threshold applied.
- Overlays the smoothed motion vector, the sampled line and the perpendicular slice.
- Readouts for speed, edge strength, grain and current preset.

**Live config**

- One config file (TOML or similar) holds every mapping value.
- The daemon watches it and applies changes immediately, without restarting audio.
- SynthDefs can be edited and reloaded live in SuperCollider.

**Presets**

- Named presets (for example "gritty", "edges-only", "soft") for quick A/B comparison.
- A hotkey or CLI command cycles presets.

**CLI**

- Examples: `acursor on`, `acursor off`, `acursor set sharpness 0.4`, `acursor preset load gritty`, `acursor radius 64`.
- Talks to the daemon over the same control interface as everything else.

**Stack**

- Python for capture, analysis, input, control and UI (numpy, OpenCV or scipy, python-osc, a light GUI toolkit).
- SuperCollider as the audio engine, driven over OSC.
- A Rust core is a possible later rewrite, not a v1 concern.

## Open explorations and parked ideas

These are recorded so they are not lost; none are v1 requirements.

- **Quadtree positioning.** The developer's intuition points at quadtrees for absolute position. One possible use: the ping drills down screen → quadrant → sub-quadrant.
- **Audible grid.** A regular or subdivided grid heard as a distinct kind of line when the cursor crosses it. Grid lines need a timbre clearly different from real edges.
- **Percentile pitch cues.** Small, subtle pitch ticks as the cursor passes 10%, 20%… of the screen width or height.
- **Mono position carriers.** A faint pitch for X and a separate timbre for Y, as a baseline carrier under the texture.
- **What ping does.** Locate, describe the region, speak coordinates, or a sweep; undecided until tested with eyes closed.
- **Pane and whole-screen modes.** How a large radius should sound is unknown.
- **Descriptions front.** OCR, a small vision model, and later AT-SPI text, for fast spoken summaries of what matters under the cursor.
- **Color and Timbre.** Bring back color, patchiness and other "vibe" qualities from the earlier Timbre project.
- **Edge orientation cue.** An explicit sound difference between horizontal and vertical edges beyond what motion direction gives.

Prior art to study: The vOICe ([seeingwithsound.com](https://www.seeingwithsound.com)) and EyeMusic. Both sonify a whole image as a timed sweep, which this project deliberately does not do; here the user's motion defines the read.

## Testing and first milestone

The first milestone is done when the developer can flip the cursor on, scrub across their real screen with eyes closed, and feel edges and texture.

**Acceptance criteria**

- [ ] Daemon starts at login as a systemd user service and idles near zero CPU when off.
- [ ] Global hotkey toggles the cane on and off, with distinct earcons; hold mode also works.
- [ ] Live capture of the Plasma Wayland desktop through the portal, with permission remembered across restarts.
- [ ] Global cursor position tracked on all monitors.
- [ ] Moving the mouse produces the texture voice, carrier and edge blips; stillness is silent.
- [ ] Scrubbing left and right over a vertical edge produces a clear blip each crossing.
- [ ] Radius adjustable from the default 32 px; pane and whole-screen modes selectable.
- [ ] Sharpness threshold audibly removes mild variation while keeping hard edges.
- [ ] Shake gesture fires the ping stub exactly once per shake.
- [ ] Preview window shows the filtered patch and motion vector.
- [ ] Editing the config or a preset changes the sound within a second, with no restart.
- [ ] Mouse-to-sound latency measured and under 20 ms.
- [ ] Audio on its own PipeWire stream; default hotkeys don't clash with Orca.

**Test approach**

- A set of test images and screens: gradients, grids, fine noise, text, photos, typical app windows.
- Eyes-closed sessions as the main test, with the preview window used only for debugging.
- 10-minute listening sessions to check fatigue, not just 10-second impressions.
- Get a blind or low-vision tester involved early; lifelong non-visual experience may differ from blindfolded intuition.
