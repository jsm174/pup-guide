# Guardians of the Galaxy

An original table (no ROM) with a three-screen pack built around a **bezel image layered over video**. This is the cleanest example in this collection of a static image used as a frame.

| | |
|---|---|
| Pack folder | `pupvideos/gotg_2020` |
| Started by | `PuPlayer.B2SInit "", "gotg_2020"` from the table's `PuPInit` |
| Installed option | "3 Screen FullDMD" (`Option Selected.txt` = 2). The root `.pup` files equal that option folder. |
| Backglass | screen 2 |
| ScoreView | screen 5 with children 11, 12, 13, 15, 16 |
| Topper | unused (screen 0 is Off) |
| Audio only | screens 4 and 14 (MusicOnly) |

## Screen layout

| # | Mode | CustomPos | Role |
|---|---|---|---|
| 2 | ForceBack | | Backglass. Its `screens.pup` background is a PuPFrames PNG, so it becomes the underlay. |
| 4 | MusicOnly | | Music |
| 5 | ForceBack | | FullDMD mode videos |
| 11 | Show | `5,0,0,100,100,4` | The **bezel image** plus every HUD label |
| 12 | ForcePopBack | `5,0,0,100,100` | Bonus screen |
| 13 | ForcePop | `5,0,0,100,100` | Splashes and the service menu |
| 14 | MusicOnly | | Wait music and effects |
| 15 | Show | `5,0,0,100,100` | Videos that play **under** the bezel |
| 16 | Show | `5,0,0,100,100` | Translucent score GIFs |

The sixth CustomPos value (`4`) is ignored ([chapter 3](../03-screens-and-layout.md#custompos-placing-a-screen-inside-another)).

The script sends FN 6 to screen 11 at startup, which moves it to the end of the screen order. The ScoreView then draws ([chapter 3](../03-screens-and-layout.md#draw-order)):

1. Step 1: the video of 5, the video of 15, then **the bezel image on 11**.
2. Step 2: screen 12 (bonus), above the bezel.
3. Step 3: the labels of 15, 16 and 11.
4. Step 6: screen 13, above everything.

## The bezel: a static image, not an overlay

The HUD frame isn't a pBackground overlay. Triggers E455, E456 and E457 play a 5312×2988 PNG (`PlayerSelectOverlay`, `GameOverlay`, `OverlayFrame-v2`) from a normal playlist on screen 11. A PNG from a normal playlist is a **static image** that shows whenever no video plays on that screen ([chapter 6](../06-playback-engine.md#images-in-a-normal-playlist)). Nothing ever plays a video on screen 11, so the image stays up until another image replaces it (E454 swaps in a blank one). It also survives stops: only an accepted video request would remove it.

Because screen 11 is a Show screen (step 1, like screen 15) and comes last in the screen order, the image draws **over** screen 15's video. That's what makes screen 15 an "under the bezel" video layer.

The script helper that plays on screen 15 does three things:

1. Moves screen 15 into place with FN 15 (`CP:` from a per-layout table).
2. Plays the video.
3. Sends FN 6 to screen 11 again, so the bezel stays in front.

To hide the under-video, the script shrinks screen 15 to `5,0,0,0,0` with FN 15 instead of stopping it.

> **Lesson.** An image-only Show screen plus FN 6 gives a clean "frame over video" layer with no alpha video. Labels are a different step, though: screen 15's own labels draw **above** the bezel even though its video is under it (step 3). Text that must sit under the frame has to be part of the video.

## How media is driven

- **Triggers** are `E` events from the script's `PuPEvent` (`B2SData "E"&n`).
  - `D0` plays `DMDSplash/@blank.mp4` (3 s of black) on screen 2 and sets `default.mp4` as the SetBG background of screen 5.
  - E454 resets screen 5 to `default.mp4`, clears screen 5's overlay, and replaces screen 11's bezel with a blank static image (`PupImages/blank.png`) until E455–E457 bring a frame back.
  - E501–E503, E530, E531, E536, E544, E582 and E599 play splashes on screen 13.
- **Most media is played by the script**, through its own helpers and a **script-side queue** (`QueueScene` / `RunQueue`). Each queued scene carries a hand-measured length in milliseconds. PinUP's own queue actions are never used.
- After every FullDMD play, the script sends `SetBackGround 5,1`, so the clip becomes the new resting loop ([chapter 6](../06-playback-engine.md#the-background)).
- Music comes from a playlist named `Topper` played on screens 4 and 14. The name has nothing to do with screen 0.
- FN 6, 11, 15 and 22 are the only custom functions used.

## The HUD

- About 100 labels on screen 11, on three pages: gameplay, high-score tables, and high-score entry.
- During the bonus, the script switches screen 11 to page 2 (the high-score page), so the page-1 gameplay labels disappear. The bonus plays on ForcePopBack screen 12 (step 2), which is above the bezel image but **below every label** (step 3). Switching pages is how the pack keeps the gameplay text off the bonus video.
- Image labels are created **first**, "so text overlaps the image". `PlayerSelect` images use `zback:1` to sit just above pBackground ([chapter 11](../11-label-properties.md#order)).
- Screen 16 has a single `ScoreAnimation` GIF label whose translucency is set with FN 22 (overlay alpha 80–255).
- Eight font families, all shipped in `FONTS` and all matched by family name.

## Other techniques

- **Black backdrop for menus.** The service menu loops `PupVideos/blank.mp4` (an all-black video) on ForcePop screen 13 and draws page-3 labels over it. It first plays a fully transparent `PupOverlays/clear1.png` to reset that screen's pBackground.
- **Blackout by overlay.** `PuPOverlays/black.png` on screen 11 during high-score entry, then `clear1.png` to undo it.
- **Deliberately missing names.** `Clear.png`, `clear.png` and `BumperPopClear.png` don't exist on purpose: they are the "clear by missing image" idiom ([chapter 10](../10-labels.md#image-captions)). The pack mixes it with the real transparent `clear1.png`.
- **Negative playlist rest.** Every playlist has `RestSeconds -1`, which means no rest ([chapter 4](../04-playlists.md#playlistspup)). The same value in a trigger row would be a 1-second delay ([chapter 5](../05-triggers.md#what-happens-when-a-row-fires)).

## What doesn't work in VPX

- **3 s of black at startup.** The `D0` row's `@blank.mp4` covers the backglass underlay for its full length.
- **Framework leftovers.** The PuPDMD framework plays `@vidIntro.mp4` on screen 1, which is Off, so the request is refused, and screen-1 labels are created but never drawn.
- **Duplicate playlist.** `PupBonus` and `PuPBonus` are the same folder for VPX; the second row is skipped and logged.
- **Version check skipped.** The script reads `GetRoot & "\gotg_2020\version.txt"` **before** `B2SInit`. With no global PUP folder set, `GetRoot` is empty at that point ([chapter 8](../08-script-api.md#getroot)), so the check is skipped. It happens to leave the right defaults.
- **Executables** (`PuPDesktopPunch.exe`, `sQRCode.exe`, `sToken.exe`) never run ([chapter 2](../02-pack-files.md#things-vpx-ignores)).
- **Memory.** Each 5312×2988 RGBA frame is about 63 MB once decoded. A frame at the window's real size would do.

## Lessons

- A static image on a screen that never plays video is a permanent frame. Order it with FN 6, and remember that labels always come after video.
- Don't read `GetRoot` before the pack is started.
