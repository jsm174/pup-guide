# Indiana Jones (Stern)

A PinMAME table with a **capture-driven** pack: 280 `D` rows tied to DMD frames in `PupCapture`, plus `D0`. The table script doesn't call PUP at all.

| | |
|---|---|
| ROM and pack folder | `ij4_210` (SAM, 128×32) |
| Started by | PinMAME, automatically ([chapter 1](../01-concepts-and-setup.md#a-automatically-from-pinmame)) |
| Installed option | "Option 2 – BG and DMD seperate" (spelled that way) |
| Backglass | screen 2 with children 11 and 12 |
| ScoreView, Topper | not PUP: the DMD is VPX's own PinMAME DMD |

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForceBack | | Root; holds the frame |
| 11 | ForceBack | `2,5.99,5.21,87.84,87.84` | Scene layer; background `BG/bg.mp4` |
| 12 | ForcePopBack | same rect | Callout layer over 11 |

Draw order ([chapter 3](../03-screens-and-layout.md#draw-order)): the video of 11 (step 1), the video of 12 (step 2), then screen 2's labels (step 3). `D0` puts `rhamen4.2.png` on screen 2's pBackground: a 1920×1080 frame with a transparent center, so the frame sits over both video layers.

> **Lesson.** Two screens with the same rectangle, a ForceBack scene layer and a ForcePopBack callout layer, let callouts play over scenes without fighting the scenes' priority ladder.

## How media is driven

- **Triggers:** 283 active rows: `D0`, 280 capture triggers, plus `W24` and `W29` (the outlane switches), which play a shuffled drain playlist.
- **Captures:** 290 BMPs, every D number has a capture, and 289 of them draw a magenta rectangle around the part that must match, so the score and other changing parts outside it are ignored ([chapter 5](../05-triggers.md#dmd-frame-triggers)).
- **Actions:** 199 SkipSamePrty, 83 Normal. No Loop, SetBG or Stop.
- **Priority ladder:** attract 1–2, skill shot 3, progress 4, story scenes 5, multiball and crystal-skull jackpots 6, other jackpots and drains 7, match and tilt 8–9, game over 10.
- **Layer split:** scenes go to 11; jackpots, "shoot the flashing", letter scoring and the map bonus go to 12.
- **Volume 1** on almost every row (about 1 %), so the ROM keeps the soundtrack. Drains, tilt and danger use 50; crystal-skull jackpots 100.
- **Trigger rest** on repeatable callouts: 3 s for jackpots, 20 s for snakes, ants and rats, 60 s for attract.

### Normal versus SkipSamePrty at the same priority

The story scenes use Normal rows at priority 5, from playlists with a 30-second rest. That gives them a subtle behavior ([chapter 6](../06-playback-engine.md#the-rest-gate)):

- A scene from the **same** playlist that arrives while another priority-5 scene plays is refused while the playlist rests.
- A scene from a **different** playlist at the same priority replaces the current one.
- On an idle screen, any positive priority skips the rest and plays.

So a sequence within one playlist behaves like SkipSamePrty, while a scene from another chapter cuts in.

## What doesn't work in VPX

- **A duplicate capture ID.** Next to `111.bmp` (the "shoot scoop" frame) the pack has `233346391 Cap_111.bmp`, a "SUPER JACKPOT" frame. VPX takes the digits right before `.bmp` as the ID, so both are ID 111, and only one is kept. D111 may then fire on the wrong frame, or never.
- **A missing file.** The "shoot arrows" row names `raidersflashingarrow.mp4`, which doesn't exist; the row is dropped at load.
- `ExactColorMatch.txt` is ignored. This pack's captures use clean 16-level shades, so dot-shade matching gives the same result.
- `balllock` is an empty playlist; `PuPShapes` and `Shapevideos` are listed but never triggered.

## Lessons

- Name captures with the number only.
- Rows with an empty PlayList are comments even when Active is 1, and this pack has several ([chapter 5](../05-triggers.md#columns)).
- Volume 0 and volume 1 both play at 1 % in VPX: neither is mute ([chapter 7](../07-volume-ducking-sounds.md#the-volume-model)).
