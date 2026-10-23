# Die Hard Trilogy

An original table (no ROM) built on the PuPDMD framework, with the whole show on the backglass: backglass art with a **cut-out** for the DMD, a per-player HUD driven by triggers, backglass lights that follow the table's GI, and a skill-shot mini-game made of labels.

| | |
|---|---|
| Pack folder | `pupvideos/DieHard` |
| Started by | the PuPDMD framework's `PuPInit`, called from `Table1_Init` after `LoadEM`: `PuPlayer.B2SInit "", "DieHard"` |
| Installed option | "Option 2 - FullDMD on Backglass" |
| Backglass | screen 2 with children 5 and 12 |
| ScoreView | none: screen 5 is inside 2 and there's no screen 1 |
| Audio only | screens 4 (music) and 6 (callouts) |

`Table1_Init` also calls `LoadEM`, which starts a B2S controller for the game `diehard`. Pack folders are found case-insensitively, so whichever of the two starts first loads `DieHard`, and the other finds the same game already loaded ([chapter 1](../01-concepts-and-setup.md#b-from-the-script-with-b2sinit)).

The script's version check parses `GetVersion` and requires at least 1.5. VPX reports a 2.x version, so it passes ([chapter 8](../08-script-api.md#getversion)).

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForceBack | | Backglass: only its pBackground (the backglass art) |
| 4 | MusicOnly, vol 70 | | Music |
| 5 | ForceBack | `2,20.3,40.6,59.5,59.5` | The FullDMD: video and every label |
| 6 | MusicOnly, vol 70 | | Callouts |
| 12 | ForcePop | `2,20.3,40.6,59.5,59.5` | Full-size event videos over the DMD |

## The cut-out backglass

The backglass art is screen 2's **pBackground**, set from `PuPOverlays/DHBackGlassON-2scrn.png`. Labels draw after every non-pop video in the window ([chapter 3](../03-screens-and-layout.md#draw-order)), so this image sits **above** screen 5's video. It works only because the PNG has a transparent hole exactly where screen 5 is: measured at 20.2/40.6/59.5/59.4 %, matching the CustomPos.

The art used by the other options (`DHBackGlassON.png`) has no transparency. Swapping the files between options without the hole would cover the DMD.

> **Lesson.** If a parent's pBackground is the backglass art, cut a transparent hole for every child screen, sized from the child's CustomPos.

## How media is driven

All media comes from `D` triggers sent by the script's `PuPEvent` (`B2SData "D"&n`). `PuPEvent` exits early while the table is tilted, so a tilt silences the pack. There's no `D0` row: the script starts the backgrounds itself.

- **Film backgrounds.** D6–D9 SetBG one background per Die Hard film on screen 5.
- **Per-player HUD.** D1–D4 play `player N overlay.png` from PuPOverlays, fired with `PuPEvent(CurrentPlayer)`. A PuPOverlays play only swaps pBackground; it ignores priority and action ([chapter 4](../04-playlists.md#special-playlists)). The rows say SetBG, which makes no difference.
- **Backglass lights follow the GI.** D17/D18 put the lit or unlit art on screen 2 from the table's GI routine. Setting the same image again doesn't reload it ([chapter 10](../10-labels.md#image-captions)), so firing these on every GI change is cheap.
- **Villain select.** An instructions Loop at priority 50, then one Loop per villain at priority 52. Equal priority replaces, so scrolling villains swaps the loop. A StopPlayer at 100 ends the select.
- **Music** loops on screen 4 at priority 16, stopped by a StopPlayer at 100.
- **Two callouts** are named `-60_….mp3`, which ducks the other screens to 60 % while they play ([chapter 7](../07-volume-ducking-sounds.md#ducking)).
- **Skill shot.** D135 sets `snowscene2.png` as SetBG on screen 5. An image SetBG clears the background video and becomes the underlay ([chapter 6](../06-playback-engine.md#the-background)).
- Every playlist has `RestSeconds -1` (no rest).

## The HUD

- About 110 labels on screen 5, on 17 pages: scores, splash, a 26-letter keyboard of image labels for initials, bonus, skill shot, mystery, hurry-up, jackpot, leaderboard and win.
- Positions are written as 1920×1080 pixels and converted to percent by the framework helpers.
- Fonts Impact, PKMN Pinball, Instruction and DieHardVPX, all shipped.
- Animated GIF pop-ups use the framework's "play once and dispose" helper (`animate`, `gifloop:0`, `aniendhide`, `anidispose`).

## Techniques worth stealing

- **A mini-game made of labels.** The skill shot moves image labels with `pDMDLabelSetPos` from a 16–30 ms VPX timer and swaps to a red image inside the hit window. The script keeps x away from 0 (`if abs(tmpPos)<3 then tmpPos=3`), because position 0 means "the natural spot" for the alignment, not the left edge ([chapter 10](../10-labels.md#positioning)).
- **Forcing an image reload.** The leaderboard image is regenerated on disk, so the script first sets `".png"` (a missing image, which clears the label) and then the real path. Without that, setting the same path would show the old image.
- **Two stops on one event.** D16 fires two StopFile rows, one per screen. Rows with identical Trigger text fire together per screen ([chapter 5](../05-triggers.md#matching-rules)).

## What doesn't work

- **Defeated villains are refused.** Scrolling from a live villain (Loop, priority 52) to a defeated one (Loop, priority 12) is a lower-priority request, which is dropped ([chapter 6](../06-playback-engine.md#priorities)). The previous villain's loop stays on screen.
- **A StopFile with no file.** D131 "StopBonus" has an empty PlayFile. An empty StopFile compares against the playlist's **next** file and consumes it ([chapter 6](../06-playback-engine.md#stopping)). With five shuffled bonus clips, it rarely matches the one playing, and it advances the rotation.
- **Script event-number slips.** Three different mode-music routines fire D333 (one villain's music), so D336–D338 never play. Another routine fires the "finish" sounds instead of its own. D34 and D39 are fired but have no rows.
- **Fonts given as numbers.** Three image labels pass `pDMD` (the number 5) as the font name, which logs "Font not found". They're image labels, so nothing is lost.
- FN 2 (the FF/FO flags, sent by the framework's `pDisableLoopRefresh`) is sent but not supported, and does nothing ([chapter 9](../09-sendmsg-custom-functions.md#accepted-but-not-supported)).

## Lessons

- When a menu cycles items with Loop rows, give every item the same priority.
- Give StopFile rows a file name. An empty StopFile is a rotation step, not a stop.
- Grep the script's event numbers against `triggers.pup`; copy-pasted routines are where they drift.
