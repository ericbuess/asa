# Neon Cascade

A Magic Tiles-style rhythm game built as a **single self-contained `index.html`**.
No build step, no dependencies, no external assets — open the file and play.

Built for **iPad Mini (Safari / Chrome), portrait, touch only**.

```
open index.html          # or host it anywhere and load the URL
```

---

## What it does

Black tiles fall down four lanes in time with a synthesised 48-second track.
Tap each tile as it reaches the strike line; hold the long ones; use two
fingers when tiles land together. The track speeds up twice as it moves
from **INTRO → CHORUS → FINALE**.

- 132 tiles: 122 taps, 10 holds, 13 simultaneous (two-finger) moments
- Music is generated with the Web Audio API — lead, bass, pad and drums,
  120 BPM in A minor. The tiles *are* the melody: both come from the same
  24-bar plan, so what you tap is what you hear.

## Rules

Two rule sets, chosen on the start screen. **Chill is the default.**

| Action | Chill | Strict (the original spec) |
| --- | --- | --- |
| Tap a tile in its window | Hit — PERFECT / GREAT / GOOD | same |
| Hold past the end of a hold tile | Fine | Fine |
| Tap empty space | Streak broken | **Instant round over** |
| Tap a tile far too early | Streak broken | **Instant round over** |
| Let a tile pass the strike line | Miss, streak broken | Miss — 3 in a row ends the run |
| Release a hold early | Partial credit, streak broken | **Instant round over** |
| Finger slides out of a hold | Partial credit, streak broken | **Instant round over** |

In Chill nothing ends the run early — the song always plays to the end and the
score is the whole story. Mistakes are still expensive: an abusive player who
mashes empty space for the entire track finishes with roughly 1,200 points
against 76,960 for a clean run.

A hold you carried at least `HOLD_PARTIAL_CREDIT` (55%) of the way still banks
points, scaled by how far you got, and it is awarded *before* the combo resets
so it rides the multiplier you earned.

## Architecture

One `<script>` block, one class per concern:

| Class | Responsibility |
| --- | --- |
| `Game` | State machine (`MENU → COUNTDOWN → PLAYING → ROUND_OVER → RESULTS`), main loop, layout, input, rendering |
| `AudioManager` | Web Audio synthesis, iOS unlock, look-ahead scheduling, **the master clock** |
| `ScoreManager` | Points, combo, multiplier, accuracy |
| `ParticleSystem` / `PopupSystem` | Fixed-size object pools for hit FX and floating score text |

Canvas 2D draws the playfield; a DOM overlay draws the HUD (score, combo,
multiplier, miss pips, progress bar) and the menu / results / pause screens.

### Notable design decisions

- **The audio clock is authoritative.** Note positions are always
  `strikeY + (songTime − note.time) × pxPerSec`, so a tile's bottom edge sits
  exactly on the strike line at its beat. Frame drops move pixels, never timing.
- **Speed changes ease over 0.8 s** instead of snapping. Because `y` is derived
  from time, changing speed can never desync a note — only its on-screen
  position — so the ease is free.
- **Logical coordinate space is 1080 wide with a *derived* height**, so nothing
  is letterboxed on any aspect ratio, and fall speed is defined in *seconds of
  lead time* rather than pixels — the game feels identical on 4:3 and 16:9.
- **A tile's hit zone stretches down to the strike line.** Players aim at the
  line, not at the tile, so the sliver between an approaching tile and the line
  must not read as "empty space". The zone never grows upward, so a genuinely
  early tap still fails.
- **Holds are anchored.** A hold's rectangle sweeps down through your finger as
  it completes, so the finger is also allowed anywhere in a band around the
  strike line, as long as it stays in the lane.
- **The static backdrop is pre-rendered** into an offscreen canvas and blitted
  each frame; the per-frame work is tiles, particles and one strike-line bloom.
- **Pausing is safe**, not punishing: backgrounding the tab pauses the audio
  clock and grants any hold in flight rather than failing it.

## Tuning

Every magic number lives in the `CONFIG` object at the top of the script —
hit windows, touch tolerances, fall speed, scoring, combo tiers, the miss
threshold and the latency-compensation offset:

```js
AUDIO_OFFSET_S: 0.00,   // + = tiles arrive later relative to the audio
HIT_EARLY_S:    0.38,   // how early a tile becomes tappable
HIT_LATE_S:     0.16,   // grace after the strike line before it is a miss
TOUCH_TOLERANCE_PX: 26, // logical px of generosity around a tile
HOLD_RELEASE_GRACE_S: 0.28,  // releasing this early still completes the hold
```

What can end a run lives in `DIFFICULTIES` rather than being scattered through
the input code, so a rule set is a data change, not a logic change.

The chart is generated from `BAR_PLAN` (24 bars of `[chord, pattern, drums]`)
plus `CHORD_HITS` for the two-finger moments. Editing a bar changes the music
and the tiles together.

## Verified

Driven end-to-end in Chromium at an iPad Mini viewport (768×1024 @2x, touch):

- A robot that always aims **at the strike line** (as a human does) clears the
  song 132/132, 100% accuracy, all 10 holds — the fairness check
- Chill survives an abusive player: 1,422 empty taps and 122 misses still
  reaches SONG COMPLETE, and still scores only 1,200
- Each strict fail rule fires correctly: empty tap, three misses, early release,
  finger slip — and each of those is proven *not* to end a Chill run
- Real two-finger `Input.dispatchTouchEvent` arrives as two distinct pointers
- 60 fps median / 51 fps minimum under software rendering (SwiftShader),
  which is the floor, not the expectation, on real hardware
- Clean menu, results, retry and landscape layouts with zero runtime errors
