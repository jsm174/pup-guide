# Transformers (Stern)

A PinMAME table with a **five-zone backglass**: one main video, a text strip, two crests and two face corners, all driven by captures. It's the best example in this collection of modes switched on and off with Loop and StopFile, and of one capture feeding many screens.

| | |
|---|---|
| ROM and pack folder | `tf_180` (SAM) |
| Started by | PinMAME, automatically |
| Installed option | Option 1 "Backglass, Topper" |
| Backglass | screen 2 with children 11–15 |
| Topper | screen 0 is Off |
| Audio only | screen 4 (OST music), Off by default |

The pack's readme warns that the ROM must run **in English**: the captures are English DMD frames. Any capture-driven pack depends on the ROM's language and settings.

A second folder, `pupvideos/Muted Rom for OST Music Option/`, holds only a ROM zip. It isn't a pack; VPX only looks for `pupvideos/tf_180`. It's meant to replace the ROM by hand for the OST music option.

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForceBack | | Main video; background `Backglass_Backgrounds` |
| 11 | ForceBack | `2,29,81.2,42.4,18.8` | Text strip at the bottom; background `Text_Default.mp4` |
| 12, 13 | ForceBack | top-left and top-right corners | Decepticon and Autobot crests |
| 14, 15 | ForceBack | bottom-left and bottom-right corners | Decepticon and Autobot faces |
| 0 | Off | | Topper |
| 4 | Off | | OST music; set to MusicOnly to enable |

All six drawn screens are plain ForceBack, so their videos draw at step 1 in file order, and `D0` puts `Backglass_Overlay Only.png` on screen 2's pBackground, which draws over all of them at step 3 ([chapter 3](../03-screens-and-layout.md#draw-order)).

## How media is driven

- 315 rows, all `D`, using 213 distinct capture numbers (225 BMPs, every D number has one).
- 141 SkipSamePrty, 58 Loop, 59 Normal, 38 StopFile, 17 SetBG, 2 StopPlayer.
- 46 rows target the Off screens 0 and 4. They load, but are skipped when they fire ([chapter 3](../03-screens-and-layout.md#modes)).

### One capture, many screens

Most frames fire a single row, but the key frames (drain, game over, faction select) fire five to eight rows across screens. Rows with identical Trigger text on one screen also fire together ([chapter 5](../05-triggers.md#matching-rules)). For example, the drain frame (D2):

- StopPlayer at priority 10 on the face corners, breaking their mode loops (priority 5);
- `Ball Drained.mp4` on the text strip with Length 4;
- a drain video at priority 10 on the main screen.

### Modes as Loop plus StopFile

Each character mode follows one pattern:

1. The mode-start frame plays an intro on the main screen, and starts **Loop** rows on a face corner, the text strip and the music screen.
2. The mode-complete frame **StopFiles** exactly those files.
3. StopPlayer at priority 10 on drain is the safety net when a mode ends without its completion frame.

StopFile only stops when the current file matches ([chapter 6](../06-playback-engine.md#stopping)), so a completion frame for one mode can't stop another.

### Faction select

The "choose your side" frames loop the choose clips on every zone at priority 3, and the player's toggling replaces them (equal priority). The "selected" frames play a priority-4 clip on the main screen **and**, in the same group of rows, SetBG the faction backgrounds on the main screen, the text strip and the corners.

### A silent clip to cut the music

On tilt, the music screen plays `Empty Audio.mp3` at priority 9 with Length 6: a silent media that replaces the music, then hands back to the background.

## What doesn't work as intended

**SetBG doesn't interrupt a Loop.** A SetBG on a busy screen waits until the current media ends, and a Loop never ends ([chapter 6](../06-playback-engine.md#the-background)). So:

- After a side is chosen, the text strip keeps looping "Choose Your Side" and the corners keep their "choose" collages until a mode loop or a drain replaces them.
- The crests keep their faction-select loop through game over: the game-over SetBG rows don't reset them.
- With the OST option enabled, the "choose" music loop keeps playing after the faction is selected.

To switch a screen's resting state while a Loop runs, stop the Loop first (StopFile, or StopPlayer at a higher priority), then SetBG.

**Other notes:**

- SetBG rows with priority 20 at game over: priority does nothing for SetBG.
- The OST option needs screen 4 changed to MusicOnly **and** the muted ROM copied into the ROM folder by hand.
- Five empty playlists, including `PuPSounds` (a playlist, not the PUPSOUNDS bank).

## Lessons

- Group rows by capture number, so you can see every screen a frame drives.
- Loop + StopFile + a higher-priority StopPlayer on drain is a robust "mode on/off" pattern.
- A capture-driven pack should say which ROM language and settings it was captured with.
