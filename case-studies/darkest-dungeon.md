# Darkest Dungeon

An original table (no ROM) with a big, trigger-heavy pack: about 450 trigger rows, 84 swappable dungeon backgrounds, and separate music and callout channels. It shows good habits (real transparent clears, a mode whitelist) next to the most common pack bugs.

| | |
|---|---|
| Pack folder | `pupvideos/DarkestDungeon` |
| Started by | `PuPlayer.B2SInit "", "DarkestDungeon"`, then `LabelInit 13` and `LabelInit 2`. The labels are built 2 s later from a script timer. |
| Installed option | "3 Screen - FullDMD". The script hardcodes its own layout type and no longer reads `ScreenType.txt`, so switching options changes the `.pup` files but not the script. |
| Backglass | screen 2 with child 12 |
| ScoreView | screen 5 with children 13 and 14 |
| Topper | unused |
| Audio only | screens 4 (music) and 8 (callouts) |

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForcePopBack | | Backglass video and backglass HUD labels |
| 12 | ForceBack | `2,0,0,100,100` | "BG Overlay": only its pBackground is used, as a swappable frame |
| 5 | ForceBack | | FullDMD root, background `DMD-Overlay1.mp4` |
| 13 | ForcePop | `5,0,0,100,100` | "DMD Info": DMD labels and guide cards |
| 14 | ForcePop | `5,0,0,100,100` | Full-frame attack, attribute and jackpot videos |
| 4, 8 | MusicOnly | | Music, callouts |

On the ScoreView, 13 and 14 are both topmost pop (step 6). Screen 14 is created after 13 and moves to the front with every video, so its full-frame clips cover the screen-13 HUD while they play ([chapter 3](../03-screens-and-layout.md#draw-order)).

Screen 13 is listed **before** its parent 5. VPX creates a placeholder for 5 and replaces it when the real row comes ([chapter 3](../03-screens-and-layout.md#custompos-placing-a-screen-inside-another)). It works, but define parents first.

## How media is driven

- **Triggers** are `D` events from the script's `PuPEvent` (`B2SData "D"&n`). Most calls go through a **script-side queue** with expiry times, so a stale event doesn't play late.
- **`D0`** puts the frame `PuPOverlays/BG-Overlay.png` on screen 12 and sets the DMD background on 5.
- **Reset events:** D300 hides the backglass frame with a transparent `Blank.png` and D400 restores it; D310 clears the DMD guide card; D200 and D900 re-run the SetBG backgrounds.
- **Dungeon backgrounds:** about 84 SetBG rows on screen 2. The script stores the event for the current dungeon (`CurrentDungeonBG = "PuPevent 690"`) and replays it whenever the backglass must return to rest.
- **Side modes** loop their video at volume 0 (1 %), with the music on screen 4 as its own Loop row.
- **Priority bands on screen 2:** collects 10, items and modes 12–16, dungeon select loops 20, death blows 23, ball lost and bonus 26–27, game over 36, tilt 40, dungeon intros 41. All callouts on screen 8 use priority 1, so a new callout cuts the previous one ([chapter 6](../06-playback-engine.md#priorities)).

### A mode whitelist in `PuPEvent`

During a side mode, the script's `PuPEvent` passes only a whitelist of event numbers, and remaps the kickback event to an audio-only row so the side-mode loop isn't interrupted. This is the cleanest way to stop general play events from breaking a mode's Loop: the engine has no notion of "mode", so the script filters.

The same routine has a VBScript bug: it uses `Select Case EventNum` with `Case EventNum = 445 Or …`, which compares `EventNum` with a boolean. During the end-of-ball suppression window, every event is dropped, whitelist included. That's a script bug on every platform.

## The HUD

- Every label uses the font `"DwarvenAxe BB W00 Regular"`. The family name inside the file is `DwarvenAxeBBW00-Regular`, so it resolves through the **file-name** fallback ([chapter 10](../10-labels.md#fonts)).
- The DMD HUD (score, mode, ball) is on screen 13. The backglass HUD (hero icons, items, a heart meter, combat GIFs) is on screen 2.
- **Guide cards** are PuPOverlays PNGs played on 13: they become its pBackground, above the DMD video and below the score text.
- 90 specials use `size` rather than `fonth`, so that text comes out about 1.5× larger than the same number in `LabelNew` ([chapter 11](../11-label-properties.md#position-and-size)).

## Techniques worth stealing

- **Clear with a real transparent PNG.** Screens 12 and 13 are cleared with `Blank.png` (1920×1080, fully transparent), not a missing file. pBackground stays visible, so the next card shows normally. Compare [The Matrix](the-matrix.md), where a missing clear file hides the frame for good.
- **Ducked callouts by name.** Callout files are named `-65_…`, `-70_…`, `-50_…`, which ducks the music on screen 4 while they play ([chapter 7](../07-volume-ducking-sounds.md#ducking)).
- **Trigger rest on chatty callouts:** enemy-defeated, torch and darkness callouts use trigger RestSeconds 8, 20 and 60.
- **Separate MusicOnly screens** for music and callouts, so a callout never replaces the music.

## What doesn't work in VPX

**The backglass HUD is under the frame.** `BG-Overlay.png` is opaque across the bottom third, where the hero icons, items and heart meter are placed. Screen 2 is a pop screen, so its video **and labels** draw at step 2. Screen 12 isn't, so its labels (the frame) draw later, at step 3 ([chapter 3](../03-screens-and-layout.md#draw-order)). In VPX the frame covers that part of the backglass HUD. A layout that keeps the HUD visible puts the frame under the labels: for example the frame on screen 2's own pBackground, or the HUD labels on screen 12.

**Dead trigger rows.** These rows are dropped at load because the media doesn't exist as written:

| Trigger | Problem |
|---|---|
| D347 | `InnBackglass.mp4` (only `InnBackglass1…5` exist) |
| D306, D307 | `Apron/*.png`, only `.mp4` exists |
| D621 | playlist `AttackArbalest`, folder is `AttackArbelest` |
| D640, D632, D637, D662 | rows ask for `AncestorDefeated.mp4`, `NecromancerDefeated.mp4`, `SwinePrinceDefeated.mp4`, `DD12.mp4`; the files are `AnscestorDefeated.mp4`, `NecromancerDefeat.mp4`, `SwinePrinceDefeat.mp4`, `DD12_.mp4` |
| D902 | playlist `WizardMode` doesn't exist |
| D695, D982, D999, D903 | files missing |

**Colliding rows.** Rows with identical Trigger text on one screen all fire ([chapter 5](../05-triggers.md#matching-rules)):

- D534 has two rows: Abomination's collect also plays Flagellant's video, which replaces it. Flagellant's own collect sends 535, which is a boss-attack row.
- D248 on screen 8: the Madness callout cuts the shovel voice-over.
- D690: two SetBG rows; the second wins.

**Smaller issues:**

- FN 46 (pad mode) is sent to screen 5, which has no labels. Screens 13 and 2 are never padded.
- `LabelNew` positions such as 32.5 are rounded, and an `xAlign` of 3.7 becomes 4, which is treated as left ([chapter 10](../10-labels.md#creating-labels-labelnew)).
- `backglass` and `kickback` are listed twice in `playlists.pup`; the first rows win.
- About two dozen event numbers the script sends have no row.

## Lessons

- Clear overlays with a transparent PNG, and keep the frame screen under the screen that carries the HUD labels.
- Lint `triggers.pup` before shipping: every PlayList and PlayFile spelled exactly as on disk, and one Trigger text per screen unless you really want rows to fire together. About 15 rows here are dead or collide.
