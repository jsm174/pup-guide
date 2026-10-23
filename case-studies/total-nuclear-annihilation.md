# Total Nuclear Annihilation

A table with a small, script-sequenced pack on the ScoreView: an attract **slideshow built from SetBG swaps**, two full-size overlay screens on either side of the HUD, and GIF plates behind the text.

| | |
|---|---|
| Pack folder | `pupvideos/tna` |
| Started by | `PuPlayer.B2SInit "", "tna"`, the **first** call in `Table1_Init`, before `LoadEM` |
| Installed option | "FullDMD and Original B2S" |
| Backglass | the table's own B2S backglass: screens 2 and 6 aren't defined |
| ScoreView | screen 5 with children 11, 1, 13, 14 |
| Topper | unused |

Because PUP is started before `LoadEM`, the B2S controller's arrival is logged as "PinUpPlayer was instantiated and initialized from the table script before table init…". The same pack is already loaded, so it's harmless ([chapter 1](../01-concepts-and-setup.md#a-automatically-from-pinmame)).

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 5 | ForceBack | | Root, no media |
| 11 | ForceBack | `5,0,0,100,100` | "Underlay videos": nothing uses it |
| 1 | ForceBack | `5,0,0,100,100` | The PuPDMD screen: backgrounds and every label |
| 13 | ForcePop | `5,0,0,100,100` | Full-screen splashes, **over** the HUD |
| 14 | ForcePopBack | `5,0,0,100,100` | Drain, bonus, kill-screen videos, **under** the HUD |

Screen 1, normally a ScoreView root, is used here as a child. Any screen number can be a child.

The two pop screens are the textbook pair ([chapter 3](../03-screens-and-layout.md#draw-order)):

- ForcePopBack 14 draws at step 2: above screen 1's video, below screen 1's labels. The bonus text stays readable over the drain video.
- ForcePop 13 draws at step 6, above everything.

## The SetBG slideshow

Attract mode is twelve SetBG rows on screen 1 (`attract/part1.mp4` … `part12.mp4`, plus a rotation of hints). A VBScript timer fires them one after another, with intervals tuned to each clip, and re-fires `D0` to restart from the top.

SetBG replaces the background while the screen is idle, and it has no priority, so each step simply swaps the loop ([chapter 6](../06-playback-engine.md#the-background)). It's an easy way to sequence content that the game may interrupt at any moment: any real play cuts in, and the background comes back afterwards.

## Triggers

All 50 rows are `D` events.

- D1 sets the game background `MainBack.mp4`; D2 plays the welcome splash on 13.
- Splashes on 13 use a priority ladder: ball locks 2–3 (volume 70, Length 3), danger 6, tilt 7 (a Loop) and tilt stop 8, reactor destroyed 96–97, match 102, game over 103, reactor started 104, cancel 150.
- D107 fires three StopFile rows: one on 13 and two with identical Trigger text on screen 1, which fire together ([chapter 5](../05-triggers.md#matching-rules)).

## The HUD

- About 90 labels on screen 1, in Venera, Future Forces and LCD2, all shipped.
- No pBackground frame. The pack ships `PuPOverlays/*.png`, `PUPFrames/*` and a `Backglass/` folder, but nothing in this option uses them.
- **GIF plates.** Animated `Gif\\Textunderlay*.gif` labels with `zback:1` sit just above pBackground and act as plates behind the text ([chapter 11](../11-label-properties.md#order)).
- **Hiding by zero size.** GIF pop-ups are hidden by switching them to small blank GIFs with `width:0,height:0`. A zero-size image draws nothing.
- **Moving labels between pages** with `'pagenum'` in a special.
- Scorbit QR labels use the missing-file clear (`PuPOverlays\\clear.png`, which doesn't exist).

## Quirks it relies on

- **Rotation 0.1°.** Thirteen HUD labels are created with `LabelNew` rotation `1` (tenths of a degree), a trick meant to avoid clipping. In VPX it does nothing useful: rotated labels are clipped to the screen like any other ([chapter 3](../03-screens-and-layout.md#clipping)), and the text is just resampled through a tiny rotation. Pad captions or use FN 46 if outlines get cut.
- **Malformed specials still parse.** `"{'mt':2,}"`, `"{'mt':2,'}"`, a key with a missing opening quote, and `'height': 100.` are all read correctly. Don't count on it.
- **Duplicates.** The `GIF`/`Gif` playlist is listed twice (the second is skipped), and three labels are created twice (the first wins, [chapter 10](../10-labels.md#creating-labels-labelnew)). Both are logged and harmless here.

## What doesn't work

- **The mystery videos never play.** The changelog says events D43–D66 overlay text on `mystery.mp4`, but those rows don't exist, and the 17-video `Mystery/` playlist is referenced by nothing. About 30 events the script sends have no row.
- **An unreachable attract step.** `Case 3` appears twice in the attract timer's `Select Case`. VBScript takes the first, so part 3 is skipped in the attract loop (the post-game sequences still play it).
- **StopFile on a background.** D908 StopFiles `Hi Score.mp4`, which D103 started as **SetBG**. StopFile only checks the current media, not the background ([chapter 6](../06-playback-engine.md#stopping)), so it does nothing. Later SetBGs hide the problem.
- **GIFs moved to hidden pages.** Some GIFs are moved with `pagenum` to pages that aren't showing, and visibility only applies on the current page ([chapter 10](../10-labels.md#setting-labels-labelset)). They're probably never seen.
- The script's "use GIFs" option is never read.

## Lessons

- A pair of full-size pop children, one ForcePopBack and one ForcePop, gives "video under the HUD" and "video over everything" on the same screen.
- To stop something started with SetBG, replace it with another SetBG or clear the slot. StopFile won't see it.
- Keep `triggers.pup` and the script's event list in sync.
