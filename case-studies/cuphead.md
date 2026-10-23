# Cuphead

An original EM-style table with an almost entirely **trigger-driven** pack. Its most striking trick is the score display: twelve tiny PUP screens, each showing one digit image.

| | |
|---|---|
| Pack folder | `pupvideos/cuphead` |
| Started by | `PuPlayer.B2SInit "", "cuphead"` at **module level**, before `Table1_Init` |
| Installed option | "Option 5 - 3 Screen - DMD, BG, No Topper" |
| Backglass | screen 2 with children 6 and 7 |
| ScoreView | screen 1 (screen 5 is Off) with digit children 8–19 |
| Topper | unused |
| Audio only | screen 4 (boss music) |

`LoadEM` later starts a B2S controller for the same game; the pack is already loaded by then ([chapter 1](../01-concepts-and-setup.md#a-automatically-from-pinmame)).

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 1 | ForceBack | | DMD: per-ball background videos |
| 2 | Show | | Backglass; its pBackground is the frame (`Overlay1.png`/`Overlay2.png`) |
| 4 | MusicOnly | | Boss music |
| 5 | Off | | Unused in this option |
| 6 | ForceOn | `2,0,0,100,100` | Callout images (PuPAlphas) |
| 7 | ForceBack | `2,8,5,85,85` | Boss videos |
| 8–12, 14–18 | ForceOn | `1,x,41.4,4.6,11.5` | **One screen per score digit** |
| 13, 19 | ForceOn | `1,24,55,20,6` / `1,72.9,55,20,6` | 100k rollover badges |

On the backglass, the boss video on 7 draws first, then screen 2's frame (step 3), whose transparent window (8.4/5.0/83.3/84.3 %) matches screen 7's CustomPos. The callouts on ForceOn screen 6 draw on top ([chapter 3](../03-screens-and-layout.md#draw-order)).

## A score display made of screens

Each digit position is its own tiny screen on the DMD. On every score change, the script computes the digits and sends one event per digit:

- `E9<player><digit>` for the ones, `E8…` tens, `E7…` hundreds, `E6…` thousands, `E5…` ten-thousands;
- `E4<player+1><n>` for the rollover badge (E42n for player 1, E43n for player 2).

Rows E510–E929 play `PuPOverlays/<digit>.png` on the matching digit screen, which sets that screen's pBackground. D0 and E200 reset all of them to `0.png`. Re-sending the digit already shown is free: the same image path isn't reloaded ([chapter 10](../10-labels.md#image-captions)).

The script builds the event numbers by concatenating strings of digits computed with `/` and `INT`. That only works because `/` is floating-point division and `INT` truncates the decimals. On a system whose decimal separator is a comma, the string would no longer parse the same way.

Only three text labels exist (Players, Credits, Ball, in the shipped `Aafia` font). The rest of the HUD is images, so the pack depends on almost nothing from the label system.

## Bosses and transitions

For each boss:

- **Fight:** frame `Overlay2`, the boss video as SetBG on 7 (at volume 0, which is 1 %), the boss mp3 as SetBG on 4.
- **Defeated:** a Normal video, plus a SetBG of `Backglass.mp4`, plus `Overlay1`. These rows share one Trigger text on screen 7, so they fire together ([chapter 5](../05-triggers.md#matching-rules)): the clip plays now, and the idle loop is waiting as the background when it ends.
- **Died:** the same, plus `mute.mp3` as SetBG on screen 4 to "stop" the music by swapping in a silent loop.

The script fires "defeated" and the next boss's "fight" in the same timer tick. The later SetBG wins the background slot, frame and music, so the defeat clip plays and hands over to the next boss.

## Splashes and callouts

- Soul Lane and Perdition (E320/E321) are **SplashResume** at priority 50 on screen 7. The boss background continues where it was when they end ([chapter 6](../06-playback-engine.md#splashreset-and-splashresume)).
- Callouts E322–E326 put PuPAlphas PNGs on screen 6. E330 hides them with `blank.png`, a real 10×10 transparent image.

## Quirks it relies on

- An empty action column means Normal (151 rows).
- The B2S controller's own `E` events (tilt, match and so on use E25–E36, plus E80–E84) don't collide with the pack, which only uses E200 and above. Keep pack E numbers clear of those.
- The plunger gate fires E201 on every pass; re-setting the same SetBG doesn't restart it.

## What doesn't work

- **A misspelled playlist.** `DrKahlsRobot` in `triggers.pup`, `DrKhalsRobot` on disk. The row is dropped at load, so the "Dr Kahl's Robot died" clip (E283) never plays; its sibling rows still reset the backglass and music.
- **The option and the script disagree.** The pack's instructions say option 5 needs `usePUPDMD = false`, but the script ships with `true`. That puts the Players/Credits labels on screen 5, which is Off in option 5, and the Ball label isn't created at all. Set `usePUPDMD = false`, or install an option with a screen 5.
- E329 is sent with no row. `Dash/` and `Multiplier/` are empty folders. Screen 14 is `Show` while its sibling digits are `ForceOn` (a different draw step, invisible unless something overlaps).
- Several options tell you to delete the table's directb2s. In VPX that isn't needed: PUP takes the backglass window when screen 2 or 6 is usable ([chapter 1](../01-concepts-and-setup.md#where-screens-appear-windows)).

## Lessons

- Trigger-only packs scale well: the script just fires numbered events, and the presentation lives in `triggers.pup`.
- Ship the script default that matches the shipped default option.
- Check every PlayList name against the folders on disk.
