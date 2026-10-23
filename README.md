# Unofficial SuperParasites Extended — AU / VST3 (experiment)

The same SuperParasites engine as [Unofficial SuperParasites](https://github.com/particledots/Unofficial-SuperParasites-plugin)
(Clouds + Parasites + Beat Repeat, eight modes), with one change: **the recording buffer can be much longer**.
The original plugin stays as the faithful reference; this one is for experiments. It is a separate plug-in (code `SPEx`,
"particledots: Unofficial SuperParasites Extended"), so both can be installed side by side.
Not a Mutable Instruments product, and not made or endorsed by any of the module makers.

## Waveform display with a Position marker (version 0.5.0)
A new panel at the top of the window draws the **recording buffer** (loudest peak per column, oldest on the left, "now" on the right) and a **marker at the Position setting**:

| Mode | Marker | Shaded band |
|---|---|---|
| Granular | where the grains start reading, as time back from now | the stretch one grain plays |
| Pitch shifter / time stretch | how far back the window is read from | the window (it follows Size) |
| Looping delay, running | the delay time (Size is not used) | none |
| Looping delay, Frozen | where the loop begins | the loop (Size is its length) |
| Beat repeat | where the loop starts inside the latched slice | the loop, with the slice outlined; changes take effect at the next slice |
| Spectral madness, Spectral clouds, Resonestor | greyed out: these modes have no recording buffer | |
| Oliverb | greyed out: Position is pre-delay there, not a place in a recording | |

With **Freeze** the picture holds still (except in Beat repeat, which keeps recording while frozen, as the firmware does). The display only reads the sound memory
on the audio thread, a little at a time, and only while a plug-in window is open; the sound is bit-for-bit the same with the window open or closed (tested).

**Verified (Linux, headless; `tests/waveform_test.cpp`, run in the workflow):** where a burst of sound lands in the picture (many formats, rates, buffer lengths);
Freeze; hidden modes; output identical with display open or closed; a reader thread hammering the picture while audio runs; and, with a gliding test tone as a time stamp,
that the sound really comes from the marker (Pitch shifter, Looping delay: within a few ms) or from inside the band (Granular: grains have a lopsided window, so the heard centre sits inside the band).
**Not verified:** how the panel looks and behaves in Logic (it is drawn by code that CI compiles but nobody has seen), and the AU on the Intel Mac.

## Pitch Snap (version 0.4.0)
A new **PITCH SNAP** menu (parameter `pitch_snap`, to the right of the mode hint) settles the PITCH knob on musical notes, as in the Unofficial Spotykach and Unofficial Beads plug-ins:
* **Off (the default)** is exactly the 0.3.0 sound: the module's own detented knob curve.
* **Steps** snaps to octaves and fifths: -24, -12, -7, 0, +7, +12, +24 semitones.
* **Semitones** snaps to whole semitones.
With Steps or Semitones the PITCH readout shows the note that actually plays (a whole number such as -12 or +7), not the raw knob position. The snap is applied to the knob's value before the MIDI note
(Pitch Follows MIDI) is added, and the change glides over a few milliseconds so a step is not a click. It applies to the modes that shift pitch in semitones (Granular, Pitch shifter / time stretch, Looping delay,
Oliverb, Resonestor). In Beat repeat and Spectral clouds the firmware reads the knob position itself rather than a pitch, so the snap does not apply there. Projects saved before 0.4.0 have no such parameter and load with Pitch Snap **Off**.
There is no scale menu here: unlike Beads, this engine has no random pitch for a scale to snap.
**Verified (Linux, headless; `tests/pitch_snap_test.cpp`, run in the workflow):** the snapping function is exact over the whole range; across every knob position Steps lands only on those seven values and never moves against the knob;
in the Pitch shifter, a sine's output pitch follows the snapped value (measured within 0.1 semitone of the expected note in 7 knob/snap combinations); every mode stays finite and bounded with each setting and a sweeping knob.
**Not verified:** how it looks and feels in Logic (the menu position, the readout), and by ear in each mode.

## Limiter switch (version 0.3.0)
A new **LIMITER** button (parameter `limiter`) sits at the end of the button row. **On (the default) is exactly the 0.2.0 sound**: the soft limiter after Output Level keeps loud sounds from going past 0 dBFS.
**Off** removes it: the output is then the engine plus the Output Level gain and nothing else, so loud wet sounds can go past 0 dBFS (by tens of dB at high Output Level settings). Only two safety things
remain: NaN becomes silence and values are held inside +/-16 so a runaway can never reach the host as infinity. Use Output Level, or a limiter after this plug-in, to keep it in range.
Switching glides over about 10 ms, so there is no click. Presets and projects saved before 0.3.0 have no switch and load with the limiter **on**, i.e. they sound as they did in 0.2.0.
Why it exists: the limiter is an instant curve (no attack, no release, no look-ahead), so on very loud sounds it acts like gentle distortion, which can be heard; the switch lets you turn that off and manage the level yourself.

## Output Level (version 0.2.0)
**What was wrong.** Loading any plug-in of this family made the sound quieter than before it was loaded. With Dry/Wet at 0 (nothing but the input) I measured the output at
**-9.04 dB** relative to the input in the six modes that have a dry path (Granular, Pitch shifter / time stretch, Looping delay, Spectral madness, Oliverb, Spectral clouds), at every
host rate tested (44.1, 48, 96 kHz) and all four qualities. The cause is in the firmware code, not the wrapper: its dry/wet crossfade table starts at 0.707 (-3 dB) and its
output conversion halves the signal (x 0.5, -6 dB) before the 16-bit output. The module's analogue output stage presumably makes that up; a plug-in has none.

**The fix.** A new **OUTPUT LEVEL** knob (parameter `output_level`, -24 to +24 dB, default **+9.03 dB** = 20 x log10 (2 x sqrt 2)) sits after the engine, followed by a **soft limiter**.
With the default, a fully dry signal comes out at the level it went in. The limiter passes everything below -0.9 dBFS unchanged and bends louder peaks towards 0 dBFS, which they
never exceed; it is an instant curve (no delay, no look-ahead), a safety net against clipping rather than a mastering limiter. The gain glides (about 10 ms) so moving or automating the knob does not click.
The engine and firmware code are untouched. The stage lives in `src/OutputStage.h` and is called from `processBlock`.

**What it does not do.**
* **Beat repeat and Resonestor have no dry path in the firmware**, so "fully dry equals the input" does not apply to them. Beat repeat forces its blend to wet and the input always goes through
  the slice player (the firmware applies a 1.2 x, +1.6 dB, gain to it that the other modes' dry signal does not get); Resonestor's Dry/Wet knob is a distortion amount. Measured at the default Output Level, input -18 dBFS:
  Beat repeat **+1.57 dB** (16-bit), Resonestor **-1.61 dB** (16-bit); in the low-fi qualities Beat repeat is -0.2 dB (stereo 8-bit) and Resonestor -2.6 dB. I left those as they are: whether they should be
  trimmed to match is your call.
* **Beat repeat in Mono 8-bit (low-fi) gives no output at all** in this test (-330 dB, at every Dry/Wet setting). It is in the firmware code path and was there before this change; I have not looked into it.
* Loud dry signals are bent slightly by the firmware's own soft limit, before this stage: Looping delay, dry, input -6 dBFS: -0.06 dB; -1.9 dBFS: -0.16 dB; -0.4 dBFS: -0.22 dB.
* The firmware's output is 16-bit and is 9 dB lower than the input before this gain is applied, so the gain also raises the firmware's own 16-bit rounding noise by 9 dB (it does not add any). I have not measured or listened for it in quiet passages.
* **Presets and projects saved before 0.2.0 have no Output Level**, so they load at the +9.03 dB default and come out about 9 dB louder than they did. That is the fix, but if you
  had turned the channel up to compensate, turn it back down.

## What is different

| | Unofficial SuperParasites | This plug-in |
|---|---|---|
| Recording memory | the module's (about 1 s per channel at 16-bit) | **Buffer Length** menu: 1 s (the module's exact memory), 4 s, 10 s (default), 20 s |
| Low-fi qualities (8-bit, 16 kHz) | 4x as long as 16-bit | still 4x as long (so 10 s becomes 40 s) |
| Mono | a little over 2x stereo | still a little over 2x (the effects' workspace stays the module's size) |
| Sample rate | 32 kHz (16 kHz low-fi) | unchanged: 32 kHz (16 kHz low-fi) |
| Everything else | | identical |

Changing Buffer Length rebuilds the processor and clears the recorded audio, the same as changing Quality does.

### What you need to know about long buffers

* **The Position knob spans the whole buffer, and a long buffer starts empty.** With a 20 s buffer, Position at 0.5 reads about 10 s back, so
  until 10 s of audio has gone in you hear silence there. Let it fill, or freeze after filling. (On the module the buffer is about a second long,
  so you never notice.)
* Beat repeat picks slices back in time and used to wrap around within the one-second buffer; with a long buffer it reaches further back, so
  early slices can land on silence.
* Looping delay's maximum delay, the time-stretch window range and granular Position all scale with the buffer length.
* Memory use is small: 20 s of stereo 16-bit is about 2.6 MB.

## Source changes compared with the original SuperParasites code

1. `looping_sample_player.h` and `kammerl_player.h`: the fixed-point position arithmetic (`<< 12`) was 32-bit. It overflows once the buffer is
   longer than about 8 s (positions up to twice the buffer size are shifted left 12 bits), and the unmodified code **crashed** in my test with a
   10 s or 20 s buffer. These two files now use 64-bit integers for those positions. In Beat repeat the original added an `int32` and a `float`
   (so the sum was done in float); it is now done in 64-bit integers, which is slightly more precise. At 1 s the difference is below audibility
   but it is not bit-identical.
2. The wrapper allocates the memory and re-creates the processor when Buffer Length changes.

Everything else, including the earlier changes (the removed `debug_pin.h` include, zeroed processor memory), is as in the original plug-in.

## Install

Download `SuperParasites-Extended-macOS` from the latest passing run in the Actions tab and follow `INSTALL.txt`
(universal arm64 + x86_64, macOS 11 or later). Controls are the same as in the original plug-in; see its README.

## Verified and not verified

**Limiter switch (version 0.3.0): built on GitHub Actions (macOS runner), every step passed, run 39075610098, commit `05efc50`** (all earlier tests, the extended level tests and both mutation checks, the universal arm64 + x86_64 AU and VST3 build, signing, packaging). I can see each step's pass / fail, not its log text. Not yet loaded in Logic and not listened to.

**Limiter switch (version 0.3.0) - measured and tested here (Linux, g++, October 2026), not listened to**
* `tests/level_test.cpp`: 10485 checks, 0 failed (the 0.2.0 checks plus the switch): with the limiter on, a loud signal never passes 1.0; off, the same signal comes out at exactly the Output Level gain (within 0.0001) and
  its peak is past 1.5; a quiet signal is identical on and off; NaN and infinity never get through when off; switching glides (no step above 2 % of the whole move) and settles exactly, and switching back on gives exactly the 0.2.0 value.
* `tests/proc_level_test.cpp`: 31 checks, 0 failed, on the real processor class: the parameter exists and is on by default; a preset saved without it loads with the limiter on; with Output Level at +24 dB and a loud wet input
  (four modes) the peak is 1.0000 with the limiter on and well past 1.0 with it off, so the switch really reaches the sound.
* Mutation checks: `tests/mutation_check.sh` now breaks the stage in 7 ways (the earlier 4 plus switch ignored one way and the other, and the off-guard removed) and `tests/wiring_mutation_check.sh` in 6 ways (the earlier 3 plus
  switch not read one way and the other, and default wrong); every broken copy was caught.
* Not tested: how the switch sounds or looks in Logic, whether the button fits the panel as intended (the row is 606 of 612 pixels wide), Logic's handling of the new parameter in an existing project (the plug-in code is unchanged).

**Output Level (version 0.2.0): every step passed on the macOS runner, run 39055505535, commit `89dcc8b`** (all earlier tests, the four new level steps, the universal arm64 + x86_64 AU and VST3 build, signing, packaging). I can see each step's pass / fail, not its log text. Artifact `SuperParasites-Extended-macOS`.

**Output Level (version 0.2.0) - measured and tested here (Linux, g++, October 2026), not listened to**
* `tests/level_test.cpp`: 10,477 checks, 0 failed. Fully dry (Dry/Wet = 0) at the default Output Level, engine + stage output equals the input level within 0.1 dB in each of the six dry-path modes, at 44.1 / 48 / 96 kHz
  host, every quality the plug-in has, input -18 dBFS (and -6 dBFS at 48 kHz); worst deviation 0.064 dB. The limiter: identity up to its threshold, odd-symmetric, monotonic, slope 1 at the threshold,
  never above 1.0 for any input including infinity, NaN becomes 0. The gain is exact to 0.001 dB at -24, -6, 0, +9.03 and +12 dB; a step from +9.03 to -24 dB glides with no jump. No clipping: with a 0.95-amplitude input, all modes
  and qualities, Dry/Wet 0 / 0.5 / 1, Output Level at the default and at +24 dB, no sample exceeds 1.0; without the limiter the engine's wet peaks at the default gain would reach about 2.3 x full scale.
* `tests/proc_level_test.cpp`: 24 checks, 0 failed, on the real plug-in processor class (the same `PluginProcessor.cpp` as the AU / VST3, compiled against JUCE 8.0.6, run headless): the parameter exists and its default shows as "+9.03 dB";
  the six dry-path modes come out within 0.013 dB of the input at the defaults; setting Output Level to -24 / -9.03 / 0 / +6 dB moves the output by exactly that much (within 0.02 dB); loud input at +24 dB in every mode stays at or below 1.0;
  a preset saved without an Output Level loads at +9.03 dB (and keeps its other settings), also into an instance whose knob had been turned to 0 dB.
* **Mutation checks (the tests fail when the level is broken).** `tests/mutation_check.sh` breaks `OutputStage.h` in four ways (gain removed, default 8.0 dB instead of 9.03, wrong dB-to-gain factor, limiter removed) and the level test failed for every one
  (17, 13, 19 and 4,035 failed checks), while passing on the unbroken code. `tests/wiring_mutation_check.sh` breaks `processBlock` in three ways (stage not called, level parameter not read, parameter default 0 dB) and the whole-plug-in test failed each
  time (9, 9 and 8 failed checks); the source is restored afterwards. Both are workflow steps.
* The existing suite still passes with the new CMake targets (`engine_test`); the engine and firmware code are not touched.
* The editor source (the fifth knob in the second row) compiles against JUCE 8.0.6 with the rest of the plug-in. **How it looks is not checked: I have not seen the editor.**
* **Not verified:** how it sounds; that the level you read on Logic's meters / faders now matches; the 16-bit noise floor at +9 dB in quiet passages; the knob row in Logic; behaviour when automated; any
  match with the real module's output level; Beat repeat and Resonestor levels against a real module.

* **Verified here (Linux, October 2026), `engine_test`:** looping-delay maximum delay measures 1.04 / 4.01 / 10.01 / 20.00 s for the four
  lengths (stereo 16-bit), 10.85 s in mono, 4.09 s and 16.00 s in low-fi, so the buffers really are that long; all eight modes at all four
  lengths and three qualities stay finite, bounded and non-silent once the buffer has filled; granular, stretch, looping delay and Beat repeat read
  correctly from deep in a 20 s buffer; changing length and mode while running does not crash; everything from the original test suite
  (44.1 / 48 / 96 kHz) still passes. The plug-in sources compile against the JUCE 8.0.6 headers without warnings.
* **Not verified:** the macOS/AU build (until the GitHub run passes), loading in Logic, how it sounds, whether long buffers are *musically* useful
  in each mode, and any comparison with a real module (the module does not have this feature).
* Changing Buffer Length allocates a small amount of memory on the audio thread (the processor object, roughly 100 KB). That is a rare, deliberate
  action, but it is not strictly real-time safe.

## Planned experiments

1. Done here: long buffers at 32 kHz.
2. Next: a version running natively at 44.1 / 48 kHz instead of 32 kHz, which needs the effects' delay lengths and a few hard-coded 32 kHz constants rescaled.
3. Later, maybe: several captured layers (the Instruō Arbhar idea).

## Source and licence

Same as the original plug-in: `third_party/supercell/` is `supercell/dsp/*` and `supercell/resources.*` from `patrickdowling/superparasites`
(commit `2e9ed7c`), `third_party/stmlib/` is a subset of `mqtthiqs/stmlib` at `8ab2aaee77cbacb47b646d46d22ee5d358effe2d`. MIT per that project's README,
with MIT notices in the source files (Emilie Gillet, Matthias Puech, Julius Kammerl, Patrick Dowling); keep them if you redistribute. The wrapper code
in `src/` has no licence assigned yet. JUCE is used under its own terms (AGPLv3 / commercial).
