# King of the Hill

An original table (no ROM) whose pack has **no triggers at all**. Every video is played by the script on a single screen boxed inside the FullDMD, with a strict priority ladder.

| | |
|---|---|
| Pack folder | `pupvideos/kingofthehill` |
| Started by | `PuPlayer.B2SInit "", "kingofthehill"`, then the script reads `ScreenType.txt` and `PupSize.txt` through `GetRoot` |
| Installed option | none: `ScreenType.txt` still holds `99`, the "no .bat was run" value, so the script uses its defaults and shows a reminder on the playfield |
| Backglass | screen 2 |
| ScoreView | screen 5 with child 11 |
| Topper | none defined |

`screens.pup`, `playlists.pup` and `triggers.pup` copies also sit **next to the `.vpx`**, outside the pack. VPX only reads the files inside the pack folder, so they're ignored ([chapter 2](../02-pack-files.md#folder-layout)).

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForcePoP | | Backglass: only a pBackground image (rule cards) |
| 4 | MusicOnly | | Unused |
| 5 | ForcePop | | FullDMD: labels and the `defaultDMD.png` frame |
| 11 | ForcePopBack | `5,10.75,1.1,78.1,76.5` | Every video, boxed in the top of the DMD |
| 13 | ForcePopBack | `5,10,0,80,80` | Defined, never used |

Screen 11 is pop but not topmost (step 2). Screen 5 is topmost pop (step 6). So the frame and the score labels sit **over** the video ([chapter 3](../03-screens-and-layout.md#draw-order)). The frame's transparent hole is narrower than screen 11's box, so the frame crops the video's sides.

## How media is driven

`triggers.pup` has a single inactive row. The script's `PuPEvent` still sends `E` events, but nothing consumes them. Everything goes through `playevent` on screen 11:

| Priority | Content |
|---|---|
| (SetBG, volume 0) | `Background/DefaultBG.mp4`, which plays at 1 % |
| 15 | launch |
| 30, 35 | mode start, mode progress |
| 40 | multiball |
| 45 | lock |
| 48 | ball save |
| 60 | mode fail or win |
| 70 | mower |
| 85 | drain |
| 99 | game over |

A lower priority is **refused**, not queued ([chapter 6](../06-playback-engine.md#priorities)): a mode-progress clip (35) that arrives while a fail clip (60) plays is simply dropped. That's what this ladder wants.

- **Targeted stops.** To end a clip, the script sends `StopFile` for that exact file, at the start priority plus one (multiball 41, mower 71). StopFile only stops when the current file matches, so it can't hit the wrong clip ([chapter 6](../06-playback-engine.md#stopping)).
- **Attract carousel.** A timer steps through attract clips, all Normal plays at priority 30 that replace each other; the first step StopFiles the cycle's last clip.
- **No rest.** Every playlist has `RestSeconds -1`.

## The HUD

- One font, `Sayfull`, shipped in `FONTS`.
- All on page 1 of screen 5. The frame is `PuPOverlays/defaultDMD.png` as pBackground. Text labels use outline shadows (`shadowtype:2`).
- **Progress bars as full-frame layers.** Each indicator is an image label placed at `0,0,100,100` showing a 1920×1080 transparent PNG with only that indicator drawn. Each state swaps the whole layer (`peg0.png` … `peg4.png`). No per-element positioning is needed: the art carries the position.
- A `Star` marker is moved with `xpos`/`ypos`.
- No label animations at all.
- **Rule cards on the backglass:** `card0`–`card2.png` are played through `PuPOverlays` on screen 2, which otherwise plays nothing. The magna-save buttons cycle them. Overlay plays bypass the engine, so the swap is instant ([chapter 4](../04-playlists.md#special-playlists)).
- **Images by pack path.** `"balls\\extraball.png"` loads although `balls` isn't a listed playlist: the caption falls back to a path relative to the pack ([chapter 10](../10-labels.md#image-captions)).

## Quirks it relies on

- **Same-page splash.** The PuPDMD helper calls `LabelShowPage pDMD,1,time` while page 1 is already showing. That doesn't arm a timer; it only stops animations ([chapter 10](../10-labels.md#labelshowpage)). The script's own message queue does the real timing, so it works.
- **Double padding.** Pad mode is on, and some captions are padded by hand as well (`" FREE "`), which gives double spaces ([chapter 9](../09-sendmsg-custom-functions.md#supported)).
- **Whole numbers in `LabelNew`.** Values such as 3.8 or 21.75 are rounded. The score labels are re-set later with decimal `xpos`/`ypos`/`fonth`, which keeps decimals ([chapter 10](../10-labels.md#creating-labels-labelnew)).

## What doesn't work in VPX

- **Unlisted playlists.** `StartBBQ` (an empty folder) and `FailBoom` (no folder) aren't in `playlists.pup`, so both log "Playlist not found" when the script reaches them. A folder on disk isn't a playlist until it's listed.
- **The stop that misses.** The script's "clear events" routine calls `playstop` on screen 5, but every video plays on screen 11. Clearing the script's event queue doesn't stop the video.
- **Missing images**, all on code paths that are off: Scorbit QR images (Scorbit disabled), and `balls\\ball1-3.png` after an `exit sub`.
- `pupinit.bat` and `PuPDesktopPunch.exe` never run ([chapter 2](../02-pack-files.md#things-vpx-ignores)).

## Lessons

- A trigger-free pack is perfectly viable: one video screen, a priority ladder, and StopFile for targeted stops.
- When labels and video live on different screens, aim stops at the video screen. Keep screen numbers in named constants, so switching layouts moves labels and video together.
- List every folder you play in `playlists.pup`.
