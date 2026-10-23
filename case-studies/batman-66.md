# Batman 66

An original table (no ROM) whose backglass is a **frame with holes over four "TV" screens**, with everything driven from the script. It's also a cautionary tale about missing overlay images.

| | |
|---|---|
| Pack folder | `pupvideos/b66_orig` |
| Started by | `LoadEM` (B2S controller), then `PuPInit`, which calls `PuPlayer.B2SInit "", "b66_orig"` |
| Installed option | "Option 2 - 2 Screen - 16x9" (its triggers include a test row, E459). A leftover `Option Selected.txt` says "Option 3 - Desktop", so the folder is in a mixed state. The script matches Option 2. |
| Backglass | screen 2 with all its children |
| ScoreView | none: screen 5 is inside 2, screen 1 is Off |
| Topper | unused |

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForceBack | | Root. Its `screens.pup` background is a PuPFrames PNG (the underlay) |
| 5 | ForceBack | `2,20.2,39.7,57,56.2` | The "FullDMD" area on the backglass, with every HUD label |
| 12 | ForcePop | same rect | Bonus screen (unused) |
| 13 | ForcePop | same rect | Takeover cards and attract |
| 16–19 | Show | four rects such as `2,32,42,33.7,43.5` | Four "TV" screens |
| 4, 14 | MusicOnly | | Music, effects |

Draw order ([chapter 3](../03-screens-and-layout.md#draw-order)):

1. Step 1: videos of 2, 5 and 16–19, in file order.
2. Step 3: labels of 2, 5 and 16–19, in the same order. So **screen 5's labels, including its pBackground frame, draw above the TV videos**, and the TVs' own captions draw after that.
3. Step 6: screens 12 and 13 above everything.

## The frame with holes

The script plays `PuPOverlays` images such as `batsignal1.png`…`batsignal4.png` and per-villain art (`joker3.png`) on screen 5. They become screen 5's pBackground, stretched over the whole DMD area. Each image has **transparent holes** where the TVs are: `batsignal4.png` is open at all four TV positions, `batsignal1.png` only at the main one. Because labels come after every video in the window, the frame sits over the TV videos and the holes reveal them.

Screens 16–19 can be moved with FN 15 (`pDMDMulti2`) to rearrange the TV layout. Their captions are positioned relative to each TV's rectangle, so they follow the move. FN 15 keeps the screen's original parent and takes window percentages ([chapter 3](../03-screens-and-layout.md#custompos-placing-a-screen-inside-another)), so check the result in play.

> **Lesson.** Put the frame on the parent's pBackground and the content in Show children. The draw order composites them for you.

## How media is driven

Triggers barely matter here. The single `D0` row plays `PuPVideos/Attract1.png` as a static image with Length 1, so it disappears after a second and the PuPFrames underlay shows. The `E` rows never fire: the only `B2SData` call is reached from a framework routine whose callers are all commented out.

The script does everything: about 50 `playlistplayex`, 40 `SetLoop`, 40 `SendMSG` and 890 `LabelSet` calls.

- **A `playmedia` wrapper** picks a volume per channel, mutes the side TVs, sets SetBG on the main screen, loops on the music channels, and **ducks by hand**: FN 11 lowers the other screens, and a timer restores them ([chapter 9](../09-sendmsg-custom-functions.md#supported)). "Muted" here means volume 0, which is still 1 % ([chapter 7](../07-volume-ducking-sounds.md#the-volume-model)).
- **Music** is mp3s from a folder named `Topper`, played on screen 4.
- **Four script queues**, one per TV, each entry a VBScript command plus a duration in milliseconds. The durations were measured with a batch file running `ffprobe` over the pack's videos (`getLen.bat`, shipped in the pack).
- **Takeover cards** on ForcePop screen 13: stop, play (usually loop) an opaque `blank.mp4` as a backdrop, set `PuPOverlays/tv.png`, then set the splash labels. To clear, play a transparent `clear.png` and stop the screen.

## The HUD

- Pages 1–4 on screen 5; identical "General" text pages on each TV screen; splash, match, service and multiplayer pages on 13.
- Image labels use `zback:1` to stay under the text. Animated GIFs come from size-specific folders (`anigif.small/medium/large`).
- A real transparent `clear.png` blanks an image label **without** hiding it, the opposite of the "missing file" idiom ([chapter 10](../10-labels.md#image-captions)).
- A "grow" effect is done by a script timer stepping `width`/`height`/`size`, rather than with a zoom animation.
- The `Ball` label is rotated by 309.8° (`angle` 3098) with a caption padded by trailing spaces. VPX rotates the rendered text around its center and rounds `LabelNew` size and position to whole numbers ([chapter 10](../10-labels.md#creating-labels-labelnew)).
- Every font named exists in `FONTS` and matches by family.

## What doesn't work in VPX

- **The frame vanishes once the Bat Pole sequence starts.** That sequence plays `PuPOverlays/catwoman.png`, `joker.png`, `riddler.png` and `penguin.png`, which don't exist (only `catwoman1.png`…`catwoman4.png` and the like do). A missing overlay **hides** pBackground, and later valid overlays don't make it visible again ([chapter 4](../04-playlists.md#special-playlists)). The script never re-shows the HUD, so screen 5's frame stays hidden for the rest of the session.
- **The PuPDMD text layer is on screen 1**, which is Off in this option, so framework messages like "BALL SEARCH" never show, and `vidIntro.mp4` is refused.
- **`playlists.pup` noise:** duplicate `PuPVideos`, `PupFrames` and `PuPBonus` rows, rows for `FONTS` and `PuP-Pack_Options`, and a dozen empty folders.
- `pupinit.bat`, `PinUpPlayer.ini` and the option `.bat` files are ignored ([chapter 2](../02-pack-files.md#things-vpx-ignores)).

## Lessons

- With a "`name1`…`name4`" naming scheme, never ship a call to the bare name. One missing overlay hides the frame for good.
- Measuring clip lengths and queueing in the script gives exact sequencing across several screens, at the cost of keeping the lengths in sync with the videos.
