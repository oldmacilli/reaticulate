# Articulation timing measurement test

## Purpose

Measure an instrument articulation against a click reference so its musical
center-of-impact, rather than its MIDI note-on, lands on the beat. The output
is a per-articulation negative timing offset that can be compared with the
community Virtual Instrument Track Delays reference sheet.

This is a test harness only. It does not yet alter Reaticulate's playback
engine or bank format.

## Test project

The project has separate tracks for a click reference and the instrument under
test. Both receive events at identical project times. The instrument is
recorded or rendered post-FX; the click is rendered separately so its transient
is an unambiguous zero-time reference.

Each test case contains both a quarter-note and an eighth-note phrase. The
same articulation is tested in each phrase so a sample engine's adaptive
tightness, transition, or repetition behavior can be observed rather than
mistaken for one static delay.

## Articulation classes and analysis

### Short/percussive articulations

Examples: staccato, spiccato, pizzicato, marcato.

Analyze a fixed attack window from 300 ms before through 150 ms after the
click. Convert the rendered audio to a perceptual loudness envelope, then
calculate the loudness-weighted time centroid:

`center = sum(time * loudness) / sum(loudness)`.

The candidate negative timing offset is adjusted until the centroid is at the
click. First non-silent onset and envelope peak remain diagnostic fields; they
do not decide alignment.

### Long, slow, or crescendo articulations

Examples: sustains, slow attacks, crescendi, swells, legato transitions.

Do not apply the fixed short-articulation window: it can measure the later body
or swell instead of the intended attack. Derive an attack interval from the
first sustained loudness rise after each note-on, bounded by the next note or
a class-specific maximum. Calculate the centroid only inside that detected
attack interval. Record the attack duration and confidence. A low-confidence
case is reported for review rather than converted into an offset.

## Outputs

For every library, instrument, articulation, dynamic/velocity layer, and note
spacing, write a row containing:

* tested offset in milliseconds;
* onset, peak, and loudness-centroid deltas relative to the click;
* detected attack interval and confidence;
* quarter-note versus eighth-note delta;
* reference-sheet value and difference, when available;
* a result of `aligned`, `spacing-dependent`, `manual-review`, or `failed`.

## Acceptance criteria

* A short-articulation result is aligned when its centroid is within one audio
  block of the click and its onset precedes the click.
* A long-articulation result is aligned when its detected attack centroid is
  within one audio block of the click; a crescendo body must not pull that
  centroid later.
* A material quarter/eighth difference is reported as spacing-dependent rather
  than collapsed into one global delay.
* No measured value is presented as a Reaticulate recommendation until the
  separate MIDI-PDC gate has passed.

## Scope boundaries

The harness uses a synthetic click and an arbitrary loaded instrument. It does
not distribute vendor samples, automate a third-party sampler UI, or assume
that the community sheet is ground truth. The sheet is a comparison baseline,
not an oracle.
