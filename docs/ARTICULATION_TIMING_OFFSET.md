# Articulation Timing Offset: development brief

## Goal

Let a Reaticulate articulation advance its musical MIDI events by a configurable
number of milliseconds, so slow-speaking articulations can sound on the grid
without moving notes in the MIDI item.

The user-facing name is **Articulation Timing Offset**.  It intentionally does
not reuse REAPER's "media playback offset" name: REAPER applies that setting to
an entire track, whereas this feature changes timing by currently active
articulation.

## Non-goals for the first release

- Changing the track's `D_PLAY_OFFSET` while playing.
- Making live input arrive before it is played.  Physics prevents this; live
  input must either bypass compensation or be buffered.
- Rewriting MIDI items by default.
- Supporting different offsets per note in a chord.

## Proposed bank syntax

Use an optional articulation attribute, stored in milliseconds:

```ini
//! c=legato i=legato o=note:24 delay=-80
1 10 Slow Legato
```

`delay` is signed and defaults to `0`.  The first implementation accepts
`-250..250` ms and reports an invalid value rather than silently clipping it.
Negative values advance an articulation.  Positive values intentionally delay
it and are useful for testing and unusual patches.

## Runtime model

1. The Lua layer parses `delay` as a normal articulation property and serializes
   it as a new articulation extension.
2. The Reaticulate JSFX has a fixed maximum look-ahead (`timing_max_ms`) per
   track, initially 250 ms.  It declares that maximum as PDC to REAPER.
3. Every MIDI event is queued.  A note-on uses the delay of the active
   articulation; its note-off retains the same delay, even if a different
   articulation becomes active in the meantime.
4. Articulation output events (keyswitches, CCs, and program changes) follow
   the timing of the articulation they activate.  Ordinary CC handling is an
   explicit first-release decision: retain normal timing unless it is part of
   activation, rather than accidentally shifting expression automation.
5. With PDC active, REAPER provides playback/render look-ahead.  The JSFX
   releases a delayed input event after `maximum - articulation_delay`, yielding
   an earlier audible result relative to the source note.

## First gate: prove PDC semantics

Before changing Reaticulate's serialized protocol, make a standalone JSFX test
effect that:

- reports a fixed 100 ms PDC delay;
- passes MIDI note-ons through after 100 ms;
- passes a marked note number through after 20 ms; and
- logs output event sample positions.

In a fresh REAPER project, render clicks produced by the two notes and measure
the resulting sample positions.  The marked note must render 80 ms ahead of
the normal note while both source note starts remain aligned.  Repeat after a
seek and during looped playback.  If REAPER does not supply usable MIDI PDC
look-ahead for this configuration, stop here and choose the offline "bake MIDI
timing" alternative below.

## Data and code changes after the gate

| Area | Change |
| --- | --- |
| `app/reabank.lua` | Validate and normalize `Articulation.delay`. |
| `app/rfx.lua` | Add `ARTICULATION_EXTENSION_TIMING`; include it in the bank-to-JSFX sync. |
| `jsfx/Reaticulate.jsfx` | Add protocol constants, extension storage/LUT accessors, stable event queue, PDC declaration, and note-to-offset association. |
| Track settings UI | Expose maximum look-ahead and a recording bypass. |
| Documentation | Describe timing semantics, live-input limitation, and migration behavior. |

The current JSFX already has a versioned, extensible articulation record and a
Lua opcode path that serializes each articulation.  Adding a distinct extension
keeps existing banks and old project state compatible: missing extension means
0 ms.

## Edge cases to test

- A program change exactly on the note start.
- A note held across an articulation change.
- Same pitch retriggered before the prior note-off.
- Multiple articulation groups, cascaded `art` outputs, and destination MIDI
  channel/bus routing.
- Chase after playhead seek and after project reload.
- Playback, offline render, real-time render, loop, and tempo changes.
- Recording: compensation bypassed by default, with no timing corruption.
- Existing banks/projects without `delay` attributes.

## Fallback if MIDI PDC cannot provide look-ahead

Implement a reversible action, **Bake articulation timing into selected MIDI
items**.  It scans program changes, resolves the active articulation for each
note-on, and moves its note-on and note-off by that articulation's delay.  It
would be deterministic for playback, rendering, and recording after the fact,
but is opt-in because it changes item data.

## DAW automation harness

Use the local, file-bridge MCP from `TwelveTake-Studios/reaper-mcp` for the
first macOS DAW runs.  Its documented bridge location supports macOS and it
uses a local JSON mailbox rather than a network listener.  Install the bridge
manually and run it from REAPER's Action List; do not use an auto-start hook
until the proof-of-concept is stable.  Restrict generated projects and renders
to this checkout's `test-projects/` and `test-renders/` directories.  Never
point an experimental bridge at a production REAPER project.

`danishaft/reaper-mcp` is a promising alternative because it has a narrower,
safety-oriented default tool profile and a usable CLI, but its own README says
macOS live-DAW acceptance is still pending.  Keep it as a secondary option,
not the first macOS test harness.
