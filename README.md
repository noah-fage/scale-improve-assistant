# Scale Improv Assistant

**MUMT 306 - Music Technology | McGill University | Noah Fage | April 2026**

A [plugdata](https://plugdata.org/) patch that lets you improvise melodies that always sound in key, with an auto-generated chord pad playing underneath. Select a root note and scale type, play on the on-screen keyboard, and every note is automatically snapped to the chosen scale - no wrong notes possible. A chord pad responds to each melody note by triggering the appropriate triad, and a potentiometer connected to an Arduino controls the reverb wet/dry amount in real time.

---

## Demo

[Watch demo video](./demo.mp4)

---

## Screenshots

> *Patch screenshots coming soon.*

---

## Objectives & Motivation

As a pianist, improvisation has always been a core part of how I practice and perform. I've spent years studying theory to get better at soloing - learning the modes, understanding how harmony sits underneath a melody. The two scales I reach for most are **Dorian** and **Pentatonic**.

Dorian is my favorite mode because of its raised 6th, which gives it a bright, sophisticated quality that pure natural minor lacks - it shows up constantly in jazz and funk. Pentatonic was the first non-standard scale I found on my own during COVID, just playing around and noticing that five specific notes always sounded bluesy and groovy no matter what I did with them.

When I listen back to recordings of my playing, I sometimes catch small errors - notes slightly outside the key that most people wouldn't notice, but I do. I wanted to build something that eliminates that problem entirely: a tool where no matter what you press, you're always in the right scale, with supporting harmony underneath. The goal was to let a player focus purely on feel and expression rather than on theory in the moment.

---

## Signal Flow

```
[keyboard]
    │
[unpack f f]
    │                   └── velocity ──────────────────────────────────────────────┐
    │                                                                               │
[pd quantizer] ◄── root (hradio) ◄── scale type (hradio)                          │
    │                                                                               │
    ├── quantized MIDI note ──► [mtof] ──► [osc~] ──► [*~] ◄── [adsr~ 10 0 1 300] ┘ (MELODY)
    │
    └── scale degree ──► [pd chord_lookup] ◄── root ◄── scale type
                                │
                        [unpack f f f]
                         │      │      │
                      [mtof] [mtof] [mtof]
                         │      │      │
                      [osc~] [osc~] [osc~]    ← each with [adsr~ 5 200 0.5 500] × [*~ 0.3]
                         │      │      │
                         └──────┴──────┘
                                │
                        [+~] summing chain
                                │
                            [rev3~]
                         dry ──[*~]   wet ──[*~]
                                │             │
                           [+~] FINAL MIX
                                │
                             [dac~]

Reverb wet/dry controlled by:
  [vslider] (GUI)  OR  Arduino potentiometer via [comport 9600] → [/ 1023]
```

---

## Modules

### Module 1 - Synth Voice

The core monophonic synthesizer. The on-screen `[keyboard]` outputs MIDI note and velocity, which are separated by `[unpack f f]`. Pitch goes through `[mtof]` and into `[osc~]`, shaped by `[adsr~ 10 0 1 300]` and multiplied together via `[*~]`, then sent downstream to the mixer.

### Module 2 - Scale Quantizer (`[pd quantizer]`)

Intercepts the raw MIDI note number before it reaches `[mtof]` and snaps it to the nearest valid note in the chosen scale. The logic:

1. Subtract the root offset to get a relative pitch
2. Extract pitch class using `[% 12]`
3. Separate the octave using `[/ 12]` → `[int]` → `[* 12]`
4. Route the pitch class through one of four mapping subpatches (`[pd map_major]`, `[pd map_minor]`, `[pd map_dorian]`, `[pd map_pent]`) selected via `[spigot]` / `[== N]` logic
5. Each mapping subpatch uses `[sel 0 1 2 ... 11]` and 12 message boxes to output a corrected pitch class and scale degree
6. Reconstruct the full MIDI note by adding root and octave back

The user selects root (C–B, 0–11) and scale type (major, natural minor, Dorian, pentatonic) using two `[hradio]` selectors.

### Module 3 - Chord Pad Generator (`[pd chord_lookup]`)

When a melody note is played, the quantizer's second outlet emits the scale degree (0–6). The chord lookup subpatch routes that degree through one of four triad subpatches (`[pd triad_major]`, `[pd triad_minor]`, `[pd triad_dorian]`, `[pd triad_pent]`), each of which outputs a set of three semitone intervals above the current melody note. Three separate pad synth voices (each with `[mtof]`, `[osc~]`, `[adsr~ 5 200 0.5 500]`, `[*~]`, and `[*~ 0.3]` for reduced gain) play those notes simultaneously beneath the melody.

**Example triad intervals (major scale):**

| Degree | Intervals |
|--------|-----------|
| 0      | 0, 4, 7   |
| 1      | 0, 3, 7   |
| 2      | 0, 3, 7   |
| 3      | 0, 4, 7   |
| 4      | 0, 4, 7   |
| 5      | 0, 3, 7   |
| 6      | 0, 3, 6   |

### Module 4 - Reverb

The combined melody and chord signals pass through `[rev3~]` before the output. A dry/wet crossfade is implemented using complementary multipliers: wet gain = x, dry gain = 1−x via `[expr 1-$f1]`. Both the GUI `[vslider]` and the Arduino potentiometer feed this control.

### Module 5 - Arduino Integration (in progress)

The Arduino side of the integration is fully implemented and tested. A 10kΩ potentiometer connected to an Arduino Uno (pin A0) sends analog readings over serial at 9600 baud via `Serial.println()`, and `[comport 9600]` in plugdata successfully receives the incoming byte stream (confirmed via `[print]` in the Pd console).

The remaining step is finalizing the ASCII parsing chain on the Pd side. Because `Serial.println()` transmits values as ASCII text rather than raw integers, a value like `538` arrives as the byte sequence `53 51 56 13 10`. Converting this cleanly to a normalized float for the reverb control requires an additional parsing stage (`[fromsymbol]` / `[/ 1023]`) that is still being refined. Once complete, the potentiometer will replace the GUI slider as the live reverb controller.

**Arduino sketch:**
```cpp
void setup() {
  Serial.begin(9600);
}

void loop() {
  int val = analogRead(A0);
  Serial.println(val);
  delay(50);
}
```

**Potentiometer wiring:** outer pins to 5V and GND, middle pin (wiper) to A0.

---

## Challenges

**Scale quantization logic** was the most involved part. The subpatch has to handle every possible incoming pitch class (0–11) and snap it to the nearest scale degree, then reconstruct the full MIDI note by adding back the octave and root offset. Getting the modulo arithmetic right and keeping all four mapping subpatches consistent required careful debugging.

**Chord voicing lookup** was also non-trivial - separate triad tables were needed for each scale type, and the degree signal had to route correctly through a `[spigot]`-based switching system rather than a simpler but less reliable `[route]` approach.

**Arduino serial parsing** proved the trickiest integration challenge. `[comport]` outputs raw ASCII byte sequences rather than clean integers - a value like `538` arrives as the bytes `53 51 56 13 10`. Converting that stream into a usable float required understanding the exact byte format and building an appropriate parsing chain on the Pd side.

**`[reverb~]` does not exist** in plugdata - discovering mid-build that the object named in the original design plan wasn't a real built-in led to switching to `[rev3~]` from the ELSE library, which required re-routing the stereo outlets and adjusting the wet/dry wiring.

---

## Future Work

- Add MIDI keyboard support (physical instrument instead of on-screen keyboard)
- Add more scale types: Phrygian, Lydian, Mixolydian, harmonic minor
- Arpeggiator mode for the chord pad
- Velocity-sensitive chord voicing (softer touch = simpler voicing)
- Smooth the Arduino control signal using `[lop~]` to eliminate any stepping artifacts
- Stereo reverb using both outlets of `[rev3~]`

---

## Patch Files

- [`finalproject.pd`](./finalproject.pd) - main patch (open this in plugdata)

All subpatches (`pd quantizer`, `pd chord_lookup`, `pd map_major`, etc.) are embedded within the main patch file.

---

## Requirements

- [plugdata](https://plugdata.org/) (tested on latest stable)
- [ELSE library](https://github.com/porres/pd-else) - required for `[rev3~]` and related objects (install via plugdata's package manager)
- For Arduino integration: Arduino IDE, Arduino Uno, 10kΩ potentiometer

---

## Citations

- Puckette, M. (1996). Pure Data. *Proceedings of the International Computer Music Conference*. https://msp.ucsd.edu/Publications/icmc96.pdf
- Porres, A. T. (2023). *ELSE - a library for Pure Data*. https://github.com/porres/pd-else
- plugdata team. *plugdata - a visual programming environment based on Pure Data*. https://plugdata.org/
- Arduino. *Arduino Uno reference*. https://docs.arduino.cc/hardware/uno-rev3/
- Scavone, G. P. (2026). *MUMT 306: Music Technology*. McGill University. https://caml.music.mcgill.ca/~gary/306/
