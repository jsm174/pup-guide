# The Flintstones

A PinMAME table whose backglass is a **four-layer cake**, all driven by `D0` and DMD captures: a colored underlay, a video window, a frame, and animated text masked by a PuPShapes image.

| | |
|---|---|
| ROM and pack folder | `fs_lx5` (WPC) |
| Started by | PinMAME, automatically |
| Installed option | Option 1 "PUPSHAPES_BG, DMD, Topper" |
| Backglass | screen 2 with children 7, 11 and 12 |
| Topper | screen 0 |
| ScoreView | not PUP |

## The layer cake

| # | Mode | CustomPos | Layer |
|---|---|---|---|
| 11 | ForceBack | `2,0,0,100,100` | Full-glass color **underlay**, background `Underlay_Default.mp4` |
| 7 | ForcePopBack | `2,21.44,3.11,55.19,73.67` | The main event **video window** |
| 2 | ForceBack | | Root; its pBackground is the **frame** `Overlay_Default.png` |
| 12 | ForcePop | `2,0,0,100,100` | **Animated text**, through a PuPShapes mask |
| 0 | ForceOn | | Topper |

Back to front ([chapter 3](../03-screens-and-layout.md#draw-order)):

1. Step 1: screen 11's underlay video.
2. Step 2: screen 7's video. It's a pop screen, so when it's idle nothing is drawn and the underlay shows through.
3. Step 3: screen 2's frame, transparent across most of the glass with an opaque band at the bottom.
4. Step 6: screen 12's text videos, above everything.

### Text over the frame with a shape mask

`D0` plays `PuPShapes/PupShape.bmp` on screen 12. Wherever the shape image has the same color as its own top-left pixel, the screen's video is made transparent ([chapter 6](../06-playback-engine.md#masks)). This shape covers everything except the bottom band, so screen 12's full-screen text videos only show as a banner across the frame's opaque bottom band. That gives "text on the frame" from ordinary videos, without alpha video.

The mask applies to video only and stays until another shape replaces it.

## How media is driven

- 264 rows: 255 `D` rows and 9 switch rows (`W11` launch, two rollovers, and the three "D-I-G" lanes).
- 95 captures; every D number has one.
- Almost all rows are Normal; 26 SkipSamePrty, 2 Loop.
- Many playlists are shuffled (AlphaSort 1) and used with an empty PlayFile, which plays the next clip of a list shuffled once at load ([chapter 4](../04-playlists.md#playlistspup)).
- Most rows play at 100–120: this pack is meant to be heard **with** the ROM's sound.

### One capture, four screens

The drain frame (D2) fires a drain video on 7 (priority 10), "Drain" text on 12 (15), a drain underlay on 11 (15, volume 10) and the default topper on 0 (10). Each layer reacts to the same moment.

The `BEDROCK` letters (D82–D88) go further: each letter frame plays a letter clip on 7, the cumulative `Shape_B…BEDROCK` text on 12, a night underlay on 11 and a topper clip.

### Returning the topper to default

Many rows play the topper's own background file (`Topper_Default.mp4`) at priority 10 to "return the topper to default". If the background is already the only thing playing, nothing happens. Otherwise the file plays as a **priority-10 main media** for its 8 seconds, and any topper clip below priority 10 is refused during that time ([chapter 6](../06-playback-engine.md#priorities)). It works, with that side effect. A StopPlayer row at priority 0, which always stops ([chapter 6](../06-playback-engine.md#stopping)), would return the topper to its background without blocking anything.

## What doesn't work

Eight rows are dropped at load because their files don't exist as written:

| Frame | Requested file |
|---|---|
| D30 | `Animated_Text/01_Status Report.mp4` (exists in another folder) |
| D33, D34 | `01_YABBA.mp4`, `01_DABBA.mp4` |
| D51, D52 | `01_ONE.mp4`, `01_TWO.mp4` |
| D58 | `01_SPARE.mp4` |
| D63 | `01_Thank You.mp4` |
| D94 | `Multiball_Restart/02_Replay.mp4` |

- The `Dino_Value` playlist folder contains only a `desktop.ini`, so its two rows load but play nothing: only media files join a rotation ([chapter 4](../04-playlists.md#which-files-are-in-the-rotation)).
- Ten empty playlists, and a stray `._BTTF.png` (a macOS metadata file) in PuPOverlays. Both harmless.

## Lessons

- One capture can and should feed several layers. Keep its rows together so a missing file on one layer stands out.
- A PuPShapes mask on a ForcePop screen gives text over a frame without alpha video. The mask color is whatever pixel (0,0) is.
- To return a screen to its background, stop it; don't replay the background file as a main media.
