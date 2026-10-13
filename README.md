# Unofficial SuperParasites Extended — AU / VST3 (experiment)

The same SuperParasites engine as [Unofficial SuperParasites](https://github.com/particledots/Unofficial-SuperParasites-plugin)
(Clouds + Parasites + Beat Repeat, eight modes), with one change: **the recording buffer can be much longer**.
The original plugin stays as the faithful reference; this one is for experiments. It is a separate plug-in (code `SPEx`,
"particledots: Unofficial SuperParasites Extended"), so both can be installed side by side.
Not a Mutable Instruments product, and not made or endorsed by any of the module makers.

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
