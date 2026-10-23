# Futurama

An original table (no ROM) where the **script owns all the timing**. Every trigger has the same priority and names an exact file, and the script decides what plays when, for how long, and what else happens meanwhile (light shows, ball holds, music ducking). PUP is reduced to a player.

| | |
|---|---|
| Pack folder | `pupvideos/futurama` |
| Started by | `PuPlayer.B2SInit "", "futurama"`, then `LabelInit 5`, the pages, and two start events |
| Installed option | Option 1 (identical to the root files) |
| Backglass | screen 2 |
| ScoreView | screen 5 |
| Other windows | none |

The script's version check parses `GetVersion` and requires 1.5 or later; VPX passes ([chapter 8](../08-script-api.md#getversion)). The table can also mirror its display to FlexDMD, but PUP is the primary display and the table works without FlexDMD. Music is played by VPX, not by PUP.

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForceBack | | Backglass. Its `screens.pup` background is a PNG, so it becomes the underlay. |
| 5 | ForceBack | `5,0,0,100,100` | FullDMD: every video and label |

Screen 5's CustomPos points at itself, which leaves it a root ([chapter 3](../03-screens-and-layout.md#custompos-placing-a-screen-inside-another)). There's no `D0` row: the script starts the pack by sending E779 (backglass) and E780 (idle background).

## A script-owned scheduler

- **244 `E` rows**, all on screen 5 except E779, all at **priority 25**, all Normal. The exceptions: tilt at 26, the idle background (SetBG), one Loop for the Bender multiball, and a StopFile.
- The script wraps every event in an item with a **hard-coded duration in milliseconds** and a priority, in its own priority queue.
- When an item runs, the script fires the trigger, shows text, starts a light show and ducks the VPX music for the item's duration. It never fires two PUP events within 200 ms.
- Because every trigger has the same priority, each new video simply **replaces** the previous one ([chapter 6](../06-playback-engine.md#priorities)). PUP never has to arbitrate.
- Variations rotate in the script (`RoundRobinEvent`), not through PUP's playlist rotation. Every trigger names one file.
- The durations also **hold the ball** in scoops during cutscenes. Skipping a cutscene clears the script queue and sends `playstop`.
- During the Bender multiball the script blocks every video except the multiball loop; otherwise an equal-priority clip would replace it.

`B2SData` fires on every call, so the script can send the same event repeatedly without resetting it to 0 ([chapter 5](../05-triggers.md#matching-rules)).

> **Lesson.** If the script decides timing, give every trigger the same priority, name exact files, and keep the durations in the script. Then **check that every named file exists**: a single missing clip silently desyncs a sequence.

## The HUD

- No pBackground. Each player's HUD frame is a full-screen image label (`ScoreOverlay1`…`4`, from `PuPOverlays\\Futurama_<n>P.png`), all created up front and flipped with show/hide. A `" "` caption on an image label keeps the image and only changes visibility.
- `zback:1` on the frames, icons, credits image and wheels. Each `zback` inserts the label just above pBackground, so the **last one sent ends up lowest** ([chapter 11](../11-label-properties.md#order)). The pack uses `zback:2` on one label, which behaves the same, and orders the calls to get the stacking it wants.
- Full-screen images use align 0 with position 0, the top-left natural spot ([chapter 10](../10-labels.md#positioning)).
- `'shadowstate':1,'outline':1` gives a black outline.
- **Animations:** the mode-select ship flies with `at:2` (`mlen` 1000, `len` 600000, so it flies, then holds), is cut with `stopani` and repositioned with `xpos`/`ypos`. A suicide-booth wheel spins by sending `rotate` (tenths of a degree) every other tick.

### The label text cache and page defaults

The script's `pDMDLabelSet` remembers the last text sent to each label and skips the call when nothing changed. That keeps a frequent DMD timer cheap.

There's a catch, which the table's own changelog records as a fix ("the last game's scores would not appear"): switching to a page above 1 resets its labels to their **created** visibility ([chapter 10](../10-labels.md#labelshowpage)). The attract and previous-game labels are created hidden, so after a page switch they're hidden again, and the cache won't re-send unchanged text to show them. The fix was an uncached setter for those pages.

> **Lesson.** Caching label text is good, but after `LabelShowPage` re-send (bypass the cache), or create those labels visible.

## What doesn't work

- **Missing playlist.** E700 (attract start) names the playlist `Intro`, which doesn't exist; the row is dropped at load, so attract shows only the idle loop.
- **Missing file.** E444 names `SlurmJackpot/Slurm18.mp4`, which isn't on disk. The 17th Slurm jackpot plays nothing, while the script still holds the ball for 8 seconds.
- A few events are sent with no row (mode select, two Bender jackpots), and two rows are never sent.
- **Option 2's overlay is hidden.** Option 2 moves screen 5 into the backglass and sets a frame PNG with a hole as screen 2's underlay. But the unchanged E779 row still plays the opaque `newbackglass2.png` as a static image, which draws above the underlay and hides the frame ([chapter 6](../06-playback-engine.md#images-in-a-normal-playlist)).
- When the Bender multiball ends, the script sends a StopFile and then, 200 ms later, E780 again. The stop already restarts the background, so the second event is redundant (and harmless).

## Lessons

- Equal priorities plus a script queue is the most predictable way to run a heavily scripted pack.
- Test every page above 1 for labels that should be visible after the switch.
