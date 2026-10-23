# The Matrix

An original table (no ROM) whose pack is almost entirely **driven from the script**: three trigger rows, and about 190 scripted scenes. It also shows what happens to a pack that was designed to feed a PUP screen onto the playfield.

| | |
|---|---|
| Pack folder | `pupvideos/Matrix` |
| Started by | `PuPlayer.B2SInit "", "Matrix"` from `PuPInit`, called by `Table1_Init` |
| Installed option | `Option Selected.txt` says 3 (Desktop), but the root `screens.pup` matches no option folder: it was edited by hand |
| Backglass | screen 2 |
| ScoreView | screen 5 with children 11 and 13 |
| Topper | screen 18 (through root 0, which is Off) |

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 0 | Off | | Topper root, never drawn |
| 2 | ForceBack | | Backglass static image |
| 4 | MusicOnly | | Music |
| 5 | ForceBack | | FullDMD: nearly everything |
| 11 | ForcePopBack | `5,0,0,100,100` | Tilt, service menu |
| 13 | ForcePopBack | `5,0,0,100,100` | Super jackpot, "Dodge This" and other scenes |
| 18 | ForceBack | `0,0,0,100,100` | Playfield clips |

Two layout details:

- **An Off root can still own a window.** Screen 0 is Off, but its child 18 is usable, so the Topper window shows screen 18 ([chapter 1](../01-concepts-and-setup.md#where-screens-appear-windows)).
- **Two ForcePopBack siblings swap places.** Screens 11 and 13 both draw at step 2: above screen 5's video, below screen 5's labels and pBackground. Each new play on a ForcePopBack screen sends it to the back of the order, so whichever of the two played **last** draws under the other ([chapter 3](../03-screens-and-layout.md#draw-order)).

## How media is driven

- **Three `D0` rows**, and nothing else:
  - screen 5 loops the next `Attract` file at priority 2 (the first file in byte order);
  - screen 18 plays `LDMD_Init.mp4`;
  - screen 2 shows `Backglass/Trans3ScreenLE.png`, a 3840×2160 opaque image, as a static image. The `Loop` action means nothing for an image.
- **`SceneGeneralStart`**, called about 190 times, is the heart of the script. Each call stops the screen, plays a clip and fills ten `Msg1…Msg10` labels from a small text language (`"size:text^size:text^…"`, with `I:` for a full-screen image). This keeps every scene's text next to its clip in the script.
- **A script queue** with priorities and times (`QueueScene`), the same pattern as [Guardians of the Galaxy](guardians-of-the-galaxy.md).
- **Custom functions:** FN 5 and FN 6 (screen order), FN 11 (volume), FN 12 (stop) and FN 55 with `FS:255`, which turns off the video fade on screen 5 ([chapter 9](../09-sendmsg-custom-functions.md#supported)). The ducking helper (FN 42) is called a dozen times but begins with `exit sub`.

## The HUD

- The frame is a pBackground overlay: `PuPOverlays/GameOverlay.png` or `GameOverlay2.png`.
- About 230 labels, mostly in the "Bitwise" font. Image labels are placed in 1920×1080 pixel coordinates by a helper that converts to percent and sets `'v2':1`.
- HUD backing images use `zback:1`.
- `pDMDLabelSetColorGradient` (22 calls) and animated GIFs give the look.
- The scene text sends `shadowstate:1` then `outline:1`. Keys are applied in a fixed order ([chapter 11](../11-label-properties.md#shadow-and-outline)), so the result is a **black outline**, not a drop shadow.

## Techniques worth knowing

- **The pack as a data store.** `Callouts.csv` (file, group, length) is read with `GetRoot`. The callouts themselves are VPX table sounds, played from a grouped, cancellable script queue.
- **A page swap against ForcePopBack.** The service menu switches screen 5 to page 2 "so it doesn't overlap the labels". That comment is the [draw-order rule](../03-screens-and-layout.md#draw-order) in the author's own words: a ForcePopBack child is always under its parent's labels.
- **Attract at "volume 0".** Attract videos play at `cVolBGVideo*0`, which is 1 %, not mute ([chapter 7](../07-volume-ducking-sounds.md#the-volume-model)).

## The playfield video that never arrives

Screen 18 was meant to be **captured onto the playfield**: the script sets a flasher's `VideoCapUpdate` to `"PUPSCREEN18"`, a Windows feature that copies the contents of a desktop window with that title into the flasher. In VPX:

- `VideoCapUpdate` exists only in the Windows build and is deprecated. It looks for an OS window, and the PUP plugin creates none, so nothing is captured.
- Screen 18 instead appears in the **Topper window**, if one is enabled, because its root is screen 0.
- The table also drives its own flashers with fallback pages, so the playfield isn't left blank.

> **Lesson.** Don't feed a PUP screen into the table through window capture. In VPX a screen appears in the window its root owns, and nowhere else.

## What doesn't work in VPX

- **The HUD frame disappears after the first game.** At game end the script sets `GameOverlay.png` and then immediately `PupOverlays/clear1.png` to clear it. This pack ships `clear.png`, not `clear1.png`, so the caption is a missing image, and a missing image **hides** pBackground when it was showing one ([chapter 10](../10-labels.md#image-captions)). The next `GameOverlay.png` sets the image but doesn't make the label visible again, and the script never re-shows the HUD. The same sequence runs around high-score entry.
- **Missing images.** `PuPOverlays/Shard0.png` doesn't exist, so the "instant info" shards never show. `PuPOverlays/Info1.png` is missing too, but a later call uses `InstantInfo\Info1.png`, which exists.
- **Service-menu volume bars.** `PupOverlays/ServiceVol1.png` and `ServiceVol2.png` aren't in this pack, so the bars never show.
- **Undefined screens.** Page calls on screen 1 (and, with the playfield video option off, FN 12 on screen 19) only log "Screen not found".
- **Service-menu backdrop under the frame.** The backdrop plays on ForcePopBack 11, so screen 5's pBackground frame still draws over it.
- **Ignored files.** `pupinit.bat`, `PuPDesktopPunch.exe` and a readme asking for a patched Windows player don't apply.

## Lessons

- When you clear pBackground with a name that doesn't exist, re-show the HUD afterwards, or ship a transparent PNG under that name.
- Keep a pack's `screens.pup` in one of its option folders. A hand-edited root file is invisible to anyone reinstalling an option.
