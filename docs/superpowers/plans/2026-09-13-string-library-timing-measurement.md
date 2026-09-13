# String Library Timing Measurement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Measure Spitfire Symphonic Strings and BBC Symphony Orchestra Strings against a separate click reference at quarter-note and eighth-note spacing.

**Architecture:** A ReaScript creates a click track and two MIDI test passages for a manually loaded instrument. Separate click and post-FX instrument renders are analyzed offline. Short articulations use a fixed loudness-weighted attack centroid; long/swell articulations use a detected initial attack interval so a later crescendo cannot dominate the result.

**Tech Stack:** REAPER 7.79, ReaScript Lua, Kontakt 8, Spitfire Symphonic Strings, BBC Symphony Orchestra, Python 3, NumPy/SciPy, CSV.

## Global Constraints

- Every instrument note begins at the same project time as its corresponding click.
- Test 120 BPM quarter notes and eighth notes for every library/articulation.
- Keep renders outside Git; commit only scripts, tests, small fixtures, and documentation.
- Use reference-sheet values only as comparison baseline.

---

### Task 1: Build the click-reference test project

**Files:**
- Create: tests/reaper/Build_String_Timing_Measurement_Project.lua
- Modify: tests/reaper/README.md

**Interfaces:**
- Consumes: a manually loaded instrument FX and selected articulation class: short or long.
- Produces: Click Reference, Library Quarter, and Library Eighth tracks.

- [ ] **Step 1: Write the failing track-layout assertion**

    local expected_tracks = 3
    assert(reaper.CountTracks(0) == expected_tracks, "expected click plus quarter/eighth tracks")

- [ ] **Step 2: Run the script in REAPER with no selected instrument**

Expected: it shows “Select a loaded string instrument first” and does not create a project.

- [ ] **Step 3: Implement deterministic timing**

    local function add_notes(take, beats, midi_note, velocity, length_beats)
      for _, beat in ipairs(beats) do
        local start_time = reaper.TimeMap2_beatsToTime(0, beat)
        local end_time = reaper.TimeMap2_beatsToTime(0, beat + length_beats)
        reaper.MIDI_InsertNote(
          take, false, false,
          reaper.MIDI_GetPPQPosFromProjTime(take, start_time),
          reaper.MIDI_GetPPQPosFromProjTime(take, end_time),
          0, midi_note, velocity, true
        )
      end
      reaper.MIDI_Sort(take)
    end

Use quarter beats {1, 2, 3, 4} and eighth beats {6, 6.5, 7, 7.5, 8, 8.5, 9, 9.5}. Create matching click impulses on every value.

- [ ] **Step 4: Run it for Spitfire Symphonic Strings Violins 1 - Performance.nki**

Expected: four aligned quarter events and eight aligned eighth events.

- [ ] **Step 5: Run it for BBCSO Strings Violins 1**

Expected: the same click/MIDI geometry with a separate BBCSO project.

- [ ] **Step 6: Commit**

    git add tests/reaper/Build_String_Timing_Measurement_Project.lua tests/reaper/README.md
    git commit -m "test: add string timing click fixture"

### Task 2: Render and analyze short articulations

**Files:**
- Create: tests/reaper/analyze_timing.py
- Create: tests/reaper/test_analyze_timing.py
- Create: tests/reaper/reference_string_offsets.csv

**Interfaces:**
- Consumes: isolated click.wav, isolated instrument.wav, click times, and articulation_class="short".
- Produces: CSV columns onset_ms, peak_ms, centroid_ms, spacing, library, articulation, confidence.

- [ ] **Step 1: Write failing fixed-window tests**

    def test_short_window_is_fixed():
        assert short_window(0.0) == (-0.300, 0.150)

    def test_loudness_centroid():
        assert loudness_centroid_ms(
            np.array([-0.01, 0.0, 0.01]),
            np.array([1.0, 3.0, 0.0])
        ) == pytest.approx(-2.5)

- [ ] **Step 2: Run the tests**

    python3 -m pytest tests/reaper/test_analyze_timing.py -k short -q

Expected: FAIL because the functions do not yet exist.

- [ ] **Step 3: Implement the short analyzer**

    def short_window(click_seconds):
        return click_seconds - 0.300, click_seconds + 0.150

    def loudness_centroid_ms(relative_seconds, loudness):
        if loudness.sum() <= 0:
            raise ValueError("empty attack envelope")
        return float((relative_seconds * loudness).sum() / loudness.sum() * 1000)

- [ ] **Step 4: Run the tests**

    python3 -m pytest tests/reaper/test_analyze_timing.py -k short -q

Expected: PASS.

- [ ] **Step 5: Render and analyze Spitfire Spiccato and BBCSO Spiccato**

Render click and instrument stems separately at 48 kHz for both note spacings. Analyze all click events and write median values per library/spacing.

- [ ] **Step 6: Add reference rows**

    library,instrument,articulation,reference_ms
    Spitfire Symphonic Strings,Violins 1,Spiccato,-25
    BBCSO Strings,Violins 1,Spiccato,-50

- [ ] **Step 7: Commit**

    git add tests/reaper/analyze_timing.py tests/reaper/test_analyze_timing.py tests/reaper/reference_string_offsets.csv
    git commit -m "test: analyze short string articulation timing"

### Task 3: Analyze long and swell articulations, then compare

**Files:**
- Modify: tests/reaper/analyze_timing.py
- Modify: tests/reaper/test_analyze_timing.py
- Create: tests/reaper/compare_reference.py
- Modify: tests/reaper/README.md

**Interfaces:**
- Consumes: the same WAV pairs with articulation_class="long".
- Produces: attack_start_ms, attack_end_ms, centroid_ms, confidence, reference_ms, and result.

- [ ] **Step 1: Write the failing swell-boundary test**

    def test_detected_attack_excludes_later_swell_body():
        envelope = np.array([0.0, 0.1, 0.3, 0.55, 0.58, 0.60, 0.95, 1.0])
        start, end, confidence = detect_long_attack(envelope)
        assert (start, end) == (1, 5)
        assert confidence > 0.8

- [ ] **Step 2: Run the test**

    python3 -m pytest tests/reaper/test_analyze_timing.py -k long -q

Expected: FAIL because detect_long_attack does not exist.

- [ ] **Step 3: Implement the initial-attack detector**

Use the first frame over 8% of the local envelope maximum as start. End at the first sustained slope at or below 2% of maximum per frame; never allow the later re-rising crescendo body to extend this interval. Report confidence below 0.70 as manual-review.

- [ ] **Step 4: Run the test**

    python3 -m pytest tests/reaper/test_analyze_timing.py -k long -q

Expected: PASS.

- [ ] **Step 5: Render and analyze Spitfire Performance Legato and BBCSO Legato**

Analyze quarter and eighth passages. Compare medians: absolute difference greater than one 256-sample 48 kHz block (5.333 ms) is spacing-dependent.

- [ ] **Step 6: Implement reference comparison**

    def reference_difference(measured_ms, reference_ms):
        return measured_ms - reference_ms

    def classify(quarter_ms, eighth_ms, block_ms=5.333):
        return "spacing-dependent" if abs(quarter_ms - eighth_ms) > block_ms else "aligned"

Add reference values: Spitfire Performance Legato -80 ms; BBCSO Legato -80 ms.

- [ ] **Step 7: Generate the final comparison CSV**

    python3 tests/reaper/compare_reference.py --measured /absolute/output/results.csv --reference tests/reaper/reference_string_offsets.csv --output /absolute/output/comparison.csv

Expected: rows for both libraries, both articulations, both note spacings, with any low-confidence result marked manual-review.

- [ ] **Step 8: Commit**

    git add tests/reaper/analyze_timing.py tests/reaper/test_analyze_timing.py tests/reaper/compare_reference.py tests/reaper/README.md
    git commit -m "test: compare long string timing with reference"

