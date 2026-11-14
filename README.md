# Korda

A standalone drone instrument for the Arduino Uno R4 Minima. Seven buttons pick a scale degree, a spring-return joystick picks the chord flavor on top of it, and everything comes out as exactly three MIDI notes.

| | |
|---|---|
| **Firmware** | `Korda.ino` |
| **Board** | Uno R4 Minima |
| **Output** | MIDI OUT only, channel 1, fixed velocity 100 |
| **Design spec** | [claude.ai/code/artifact/d6953bd0…](https://claude.ai/code/artifact/d6953bd0-aa92-465c-ad59-c3ef798b8803) |

![Korda faceplate diagram showing the OLED screen, the 8-direction joystick, the menu toggle button, the unused A0 pin reserved for a future DAC, the MIDI OUT jack, seven scale-degree note buttons, and the octave down/up buttons, each labeled with its Arduino pin.](docs/images/panel.svg)

*Fig. 1 — Faceplate layout. There's no rotary encoder and no MIDI IN — Korda only ever sends.*

## Overview

Korda is monophonic by design: hold one of the seven note buttons and it always outputs exactly **three** notes — the scale root, plus two more that depend on the joystick. With the stick centered that's root + 5th + octave (an open, idle voicing). Push it in one of eight directions and the second and third notes swap to that direction's chord flavor instead. Let go of the note button and everything goes silent.

There's no rotary encoder here on purpose — an earlier revision had one, but it didn't read reliably, so the joystick took over double duty instead (see [Joystick](#joystick-play-vs-menu) below). Root and mode reset to their defaults on every boot; nothing is saved.

## Pin map

| Pin | Role | Notes |
|---|---|---|
| **D2 – D8** | Note buttons 1–7 | INPUT_PULLUP, active low. Scale degrees, not fixed semitones — what they play depends on the current root and mode. |
| **D9 / D10** | Octave down / up | Steps `octaveOffset` by one, clamped to &minus;2..+2 (&plusmn;2 octaves). |
| **A0** | *Reserved, unused* | Deliberately left open — it's the R4's DAC pin, kept free for a possible future Mozzi audio-synthesis output. |
| **A1 / A2** | Joystick X/Y | Spring-return, dual-role — see below. |
| **A3** | Menu toggle | Switches the joystick (and the OLED) between Play and the root/mode menu. |
| **A4 / A5** | OLED I2C | SSD1306 128&times;64 @ 0x3C. |
| **D0 / D1** | Serial1 &middot; MIDI OUT | DIN jack, send-only — RX is unused. DIN&harr;USB merging, if you need it, lives on a separate box, not this board. |

## Joystick: Play vs. Menu

![Two compass diagrams comparing what the joystick's eight directions do in Play mode versus Menu mode. In Play, each of the eight directions selects a chord flavor, with the top position resetting to idle. In Menu, only the four cardinal directions are active — up and down step the root note, left and right step the mode — while the four diagonals do nothing, shown dashed.](docs/images/joystick-map.svg)

*Fig. 2 — The same eight directions mean something completely different once A3 toggles Menu open. In Menu, the four diagonals are simply inert.*

**In Play** (the default), all eight directions are live: pushing one selects that chord flavor; pushing the same one again returns to idle. The top push is a dedicated reset, not a flavor — it always clears back to idle rather than toggling.

| Flavor | Offsets above root |
|---|---|
| Major (W) | +4, +7 |
| Minor (E) | +3, +7 |
| Sus2 (NW) | +2, +7 |
| Sus4 (NE) | +5, +7 |
| Min7 (S) | +3, +10 |
| Add9 (SW) | +2, +14 |
| Dim (SE) | +3, +6 |
| *Idle (center / top)* | *+7, +12* |

**In Menu**, only the four cardinal pushes do anything — up/down step the root by a semitone (clamped to MIDI 60–72, C4–C5), left/right step through the seven modes. The diagonals are inert there; there's no second layer hiding behind them.

A raw stick reading has to hold steady for 20 ms before it's accepted — the spring's own return-to-center can wobble through a neighboring sector on the way back, and without that debounce, the wobble would read as a real push and silently change the flavor.

## Modes & scale degrees

The seven note buttons are scale degrees 1–7, not fixed pitches — what each one plays is `root + MODE_INTERVALS[mode][degree] + octaveOffset × 12`. The mode table is in standard church-mode order:

| # | Mode | Intervals (semitones) |
|---|---|---|
| 1 | Ionian | 0, 2, 4, 5, 7, 9, 11 |
| 2 | Dorian | 0, 2, 3, 5, 7, 9, 10 |
| 3 | Phrygian | 0, 1, 3, 5, 7, 8, 10 |
| 4 | Lydian | 0, 2, 4, 6, 7, 9, 11 |
| 5 | Mixolydian | 0, 2, 4, 5, 7, 9, 10 |
| 6 | Aeolian | 0, 2, 3, 5, 7, 8, 10 |
| 7 | Locrian | 0, 1, 3, 5, 6, 8, 10 |

Root defaults to MIDI 60 (C4) and is clamped to 60–72 (C4–C5); mode defaults to Ionian.

## Display

- **Play**: a large note name (or `--` if nothing's held) with the active flavor name underneath, and an octave indicator on the right — a boxed column of all five positions (&minus;2..+2) with the current one drawn inverted, so it's a glance, not a read.
- **Menu**: just "Menu", the current root note name, and the current mode name.

Note on/off is diffed every cycle against what's actually sounding — only the notes that changed get a Note Off or Note On; a note that's unchanged between the old chord and the new one is never retriggered.

## Build & verify

- [ ] Libraries: MIDI Library, Bounce2, Adafruit GFX, Adafruit SSD1306
- [ ] Board: `arduino:renesas_uno:minima`
- [ ] Logic self-check: uncomment `#define KORDA_SELFTEST` and watch the native `Serial` @9600 (independent of the `Serial1` MIDI output) for `PASS`/`FAIL` on the chord-offset, mode-table, and direction-wedge math before trusting a change to it

---

Keybench / UnoWeb
