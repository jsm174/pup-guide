# Big Trouble in Little China

A fairly elaborate pack for an original table (no ROM). It uses most of what the guide describes: two windows, child screens with CustomPos, alpha video, a HUD built from image labels, pages, moving score labels and script-driven media. This page walks through it end to end.

| | |
|---|---|
| Pack folder | `pupvideos/BTILC` |
| Started by | `PuPlayer.B2SInit "", "BTILC"` from the table's `PuPInit` |
| Installed option | "Desktop-VR - 3 Screen" (`ScreenType.txt` = 0) |
| Backglass | screen 2 |
| ScoreView | screen 5 with children 11 and 13 |
| Audio only | screen 4 (music) |

## The files

```
BTILC_2026_Original_1.0.0.vpx
pupvideos/
  BTILC/
    screens.pup  playlists.pup  triggers.pup
    FONTS/          albertusnovablack.ttf  "Galpon Black.ttf"
    PuPOverlays/    DefaultDMD.png  DefaultDMDA.png  EB1.png … none.png …
    Splash_Alpha/   Fireworks30.mov  RainEffect30.mov …
    Backglass/  Background/  BallSave/  Jackpots/  Mode1/ … Wizard/
    PuP-Pack_Options/   (alternate .pup sets for Windows .bat installers)
    *.bat  *.exe  pupinit.bat  ScreenType.txt  usePuPDriver.ini
```

In VPX, the `.bat`/`.exe` files never run ([chapter 2](../02-pack-files.md#things-vpx-ignores)). `ScreenType.txt` is read by the **table script**, not by VPX.

## `screens.pup`

```
ScreenNum,ScreenDes,PlayList,PlayFile,Loopit,Active,Priority,CustomPos
2,Backglass,,,0,ForceBack,,
4,Music,,,0,MusicOnly,,
5,FullDMD,,,0,ForceBack,,
11,"Spash Alpha",,,0,ForcePop,,"5,0,0,100,100,6"
13,"DMD BG Video",,,0,ForcePopBack,,"5,0,0,100,100"
```

What VPX makes of it:

| Screen | Window | Role |
|---|---|---|
| 2 ForceBack | **Backglass** root | Backglass video. |
| 4 MusicOnly | none | Audio only (music). |
| 5 ForceBack | **ScoreView** root | The FullDMD. Carries all the labels (`pDMD = 5`). |
| 13 ForcePopBack, `5,0,0,100,100` | inside ScoreView, full size | A video layer **above screen 5's video but below screen 5's labels** ([draw order](../03-screens-and-layout.md#draw-order) steps 2 and 3). |
| 11 ForcePop, `5,0,0,100,100,6` | inside ScoreView, full size | **Topmost pop**: drawn over everything, labels included. Used for alpha-channel effect videos. The trailing `6` flag is ignored. |

The resulting ScoreView stack, back to front:

```
screen 5 video  →  screen 13 video  →  screen 5 labels (HUD, scores)  →  screen 11 alpha effects
```

The volume column is empty everywhere, so every screen has a multiplier of 100.

## The option variants

The pack ships eight option folders for Windows installers. `playlists.pup` is identical in all of them. In VPX ([chapter 3](../03-screens-and-layout.md#screen-numbers)):

| Option | Difference from the active set | In VPX |
|---|---|---|
| Desktop-VR - 3 Screen | identical | as above |
| 3 Screen - FullDMD | identical | same |
| 4 Screen - BG DMD Topper Separate | adds a ForceOn screen 0 with a `D0` topper SetBG | the Topper window gets the loop |
| 3 Screen - BG and Slim DMD | screen 5 placed inside 2, no screen 13, adds a ForceOn screen 1 | screen 5 still has child 11, so it shows in **both** windows, and screen 1 is never drawn |
| 2 Screen DMD on BG no Topper | screen 5 inside 2, no 13, the `D0` row on 11 inactive | backglass shows the DMD; ScoreView duplicates it |
| VR - 3 Screen | screen 2 is ForcePopBack; adds a MiniGame screen 12 listed **before** its parent 5 | screen 12 unused by the script |
| Pinup-1.5, Pinup-2.0 | only `usePuPDriver.ini` | ignored |

The script reads `ScreenType.txt`, but every type maps to the same DMD screen, 5. Only type 2 moves the mini-game. The script's non-Windows branches (fonts, offsets) are commented out, so VPX gets the Windows sizes on every platform.

## `playlists.pup`

```
ScreenNum,Folder,Des,AlphaSort,RestSeconds,Volume,Priority
0,PuP-Pack_Options,,0,-1,100,0
0,FONTS,,0,-1,100,0
0,PuPOverlays,,0,-1,100,0
0,Backglass,,0,-1,100,0
0,Background,,0,-1,100,0
…
0,PuPOverlays,,0,-1,100,0
0,Splash_Alpha,,0,-1,100,0
…
```

- `ScreenNum` 0 is ignored. Every playlist exists on every screen ([chapter 4](../04-playlists.md#each-screen-has-its-own-copy)).
- `AlphaSort` 0 → alphabetical. `RestSeconds` −1 → no rest. Videos are always named explicitly anyway.
- `PuPOverlays` is listed twice. The second row logs `Duplicate playlist` and is skipped, which is harmless.
- `FONTS` and `PuP-Pack_Options` are listed as playlists. Nothing ever plays from them. Harmless too.
- Listing `PuPOverlays` makes it the **special overlay playlist** ([chapter 4](../04-playlists.md#special-playlists)). This pack doesn't play from it, though: it uses its images as **label captions**, where the first path component (`PuPOverlays`) is resolved as a playlist.
- `playlists-old.pup` is ignored: only `playlists.pup` is read.

## `triggers.pup`

```
ID,Active,Descript,Trigger,ScreenNum,PlayList,PlayFile,Volume,Priority,Length,Counter,RestSeconds,Loop,Defaults
3,1,,D0,2,Backglass,Backglass.mp4,,20,,,,SetBG,0
4,1,,D0,5,Background,Background.mp4,,20,,,,SetBG,0
```

Only two rows, both on the startup event `D0`: when the pack loads, screen 2 and screen 5 each get their **background loop**. The Volume column is empty, so it's the playlist volume (100), unlike a `screens.pup` background, which would be 1 % ([chapter 3](../03-screens-and-layout.md#the-default-background-columns-2-and-3)). Everything else is driven from the script.

## Starting it from the script

```vbscript
Dim pGameName : pGameName = "BTILC"

Sub PuPInit
    Set PuPlayer = CreateObject("PinUpPlayer.PinDisplay")
    PuPlayer.B2SInit "", pGameName          ' finds pupvideos/BTILC

    pupPackScreenFile = PuPlayer.GetRoot & "BTILC\ScreenType.txt"   ' GetRoot = ".../pupvideos/"
    ' ... reads the screen layout chosen by the Windows installer .bat ...

    pDMD = 5
    PuPlayer.LabelInit pDMD                 ' overlay + pBackground on screen 5
    pSetPageLayouts
    pDMDSetPage(pDMDBlank)                  ' page 0, the default page
    ...
End Sub
```

There's no ROM, so nothing loads automatically. `B2SInit` performs the full load, and `D0` starts both backgrounds.

## The HUD: fonts and labels

```vbscript
Const dmddef = "Galpon Black"
Const dmdNum = "Albertus Nova Black"
```

Both fonts are shipped in `FONTS` and found by their family names ([chapter 10](../10-labels.md#fonts)). With a Windows-only font name here, every text label would be blank.

```vbscript
Sub pSetPageLayouts
    pDMDAlwaysPAD                           ' FN 46: pad text with spaces, '~' = 2 spaces
    pupCreateLabelImageDMD "DMDOverlay", "PupOverlays\\DefaultDMD.png", 0, 0, 100, 100, 1, 0
    pupCreateLabelImageDMD "DMDOverlayAttract", "PupOverlays\\DefaultDMDA.png", 0, 0, 100, 100, 5, 0
    pupCreateLabelImageDMD "EB", "PupOverlays\\EB1.png", 0, 0, 100, 100, 1, 0
    PuPlayer.LabelNew pDMD, "Splash",  dmddef, FontSizeSplash+4, cGold, 0, 1, 1, 50, 32, pScores, 0
    PuPlayer.LabelNew pDMD, "PointsL", dmddef, FontSizeSplash+2, cGold, 0, 1, 1,  5, -20, pScores, 0
    ...
End Sub

Sub pupCreateLabelImageDMD(lName, lFilename, xpos, ypos, Iwidth, Iheight, pagenum, lvis)
    PuPlayer.LabelNew pDMD, lName, "", 50, RGB(100,100,100), 0, 1, 1, 1, 1, pagenum, lvis
    PuPlayer.LabelSet pDMD, lName, lFilename, lvis, "{'mt':2,'width':" & Iwidth & ",'height':" & Iheight & ",'xpos':" & xpos & ",'ypos':" & ypos & "}"
End Sub
```

What's going on:

- **Image HUD frames as labels.** Each frame is an image label with an empty font (fine for images), sized `width/height` 100 × 100 % of screen 5.
- **Position 0 = centered.** The frames are placed at `xpos 0, ypos 0` *without* `v2`. Because they're center-aligned, position 0 means 50 % ([chapter 10](../10-labels.md#positioning)), so each frame is **centered** and covers the whole screen. With `v2` they'd be centered on the top-left corner instead.
- **`"PupOverlays\\DefaultDMD.png"`**: VBScript doesn't escape backslashes, so the caption really contains `\\`. Label captions collapse repeated separators and accept `\` on every OS. `PupOverlays` matches the `PuPOverlays` playlist case-insensitively.
- **Pages.** The gameplay HUD frames are on page 1 (`pScores`), the attract-mode frame on page 5. `pDMDSetPage` switches between them. They're created hidden (`lvis` 0). Pages 0 and 1 don't reset visibility on a switch, but page 5 does ([chapter 10](../10-labels.md#labelshowpage)).
- **Off-screen start positions.** Labels like `PointsL` are created at `ypos -20`, above the screen, so they're out of view until an animation moves them in.

### Label styling

- **Gradients:** 41 calls send `color`, `gradstate:1` and `gradcolor`, a full-height vertical gradient, mostly white to yellow ([chapter 11](../11-label-properties.md#color-and-look)).
- **Borders:** 58 calls set outline shadows. These helpers send a `` `u` `` caption with visible 1, which applies even off-page; harmless here because the captions are empty, and empty labels aren't drawn ([chapter 10](../10-labels.md#pages)).
- **Z-order:** `zback:1` is re-sent on every HUD frame swap, keeping the frame just above pBackground and under the text. `ztop:1` keeps small mode badges above everything ([chapter 11](../11-label-properties.md#order)).
- Only move-with-fade and pulse animations are actually called; the zoom, wiggle and clone helpers appear only in comments.

## Moving score labels

```vbscript
PuPlayer.LabelSet pDMD, "PointsL", FormatScore(PointsAdjusted), 0, _
    "{'mt':1,'at':2,'len':1000,'fc':" & cWhite & ",'xps':5,'xpe':35,'yps':20,'ype':78,'tt':6,'ad':1,'af':700}"
```

- `visible` 0 → the label returns to **hidden** when the animation ends ([chapter 12](../12-label-animations.md#rules-common-to-every-animation)).
- `at:2` moves the label's top-left corner from (5 %, 20 %) to (35 %, 78 %), with cubic in-out easing (`tt:6`), over 1 s.
- `af:700` fades it out over the last 700 ms.
- `ad:1` is ignored: a new score replaces the running animation.

## Image labels used as switches

```vbscript
PuPlayer.LabelSet pDMD, "EB", "PupOverlays\\EB2.png", 1, ""     ' show 2 extra balls
PuPlayer.LabelSet pDMD, "EB", "PupOverlays\\none.png", 1, ""    ' show nothing
```

The pack ships a transparent `none.png` and swaps it in to "clear" the label. That's more reliable than an empty caption, which is **ignored** on image labels ([chapter 10](../10-labels.md#caption-rules)). The table's `pDMDHideAnimate` helper (`LabelSet …, "", 0, ""`) still works for hiding: the empty caption is ignored, but the `visible` 0 applies (when the label is on the current page).

## Media from the script

```vbscript
PuPlayer.playevent pDMD, "BallSave", "BS_Gracie.mp4", nPupVideoVolume, 55, 0, 0, ""   ' Normal, pri 55
PuPlayer.playevent pDMD, "Misc", "Extraball.mp4", nPupVideoVolume, 90, 0, 0, ""      ' finds Misc/ExtraBall.mp4
PuPlayer.playevent pDMD, "Misc", "Tilted.mp4", nPupVideoVolume, 89, 0, 3, ""          ' cut after 3 s
PuPlayer.playevent pDMD, "Background", "Background.mp4", 0, 20, 6, 0, ""                 ' SetBG
```

- The priorities (55, 89, 90) make important videos interrupt lesser ones and refuse lower ones ([chapter 6](../06-playback-engine.md#priorities)). A ball-save video (55) can't cut off an extra-ball video (90).
- `Tilted.mp4` with `seconds` 3: the tilt video ends 3 s after the request, and the background comes back ([chapter 8](../08-script-api.md#playevent)). If a priority-90 video were playing, the tilt request would be refused and the playing video left alone.
- `ResetBackglass` re-sets the background with **volume 0**, i.e. 1 % from then on.

The script also plays `EnterInitials.mp4` (type 8, SkipSamePrty) and `ComboRecord.mp4` (type 11, QueueSameP) from a `HighScore` playlist, but this pack has no `HighScore` folder or `playlists.pup` row, so those calls only log `Playlist not found`. With the folder present, `ComboRecord.mp4` (priority 92) would wait behind `EnterInitials.mp4` (same priority) instead of replacing it.

### Modes: "stop, then SetBG the mode loop"

- At mode start, the script sets the **mode loop** as SetBG at volume 125 (audible, unlike the base background at volume 0, which is 1 %), and 200 ms later plays the intro clip at priority 65. The intro plays first; the loop takes over when it ends.
- After any interruption, `CheckBackgroundVideo` stops the screen 50 ms later and sets the current mode loop as SetBG again 100 ms after that (18 call sites). Stop plus SetBG is the pack's single way back to the resting state, which makes it hard to end up on the wrong loop.
- Mode progress clips use **SkipSamePrty** at 62, so rapid shots don't cut each other off ([chapter 6](../06-playback-engine.md#skipsameprty)).
- Loops are broken with StopFile: the four skill-shot value clips loop at 60 while the ball is in the plunger lane and are StopFiled at 61; the game-over loop (95) is StopFiled when attract starts.

> **Lesson.** Keep one routine that returns a screen to its current resting loop, and call it after anything that interrupts.

### Attract mode

- Attract starts by StopFiling the game-over loop, switching to page 5 and **then** showing the attract frame. The order matters: switching to a page above 1 resets its labels to their created visibility ([chapter 10](../10-labels.md#labelshowpage)).
- After 200 ms, `Credits/CreditIntro.mp4` plays at priority 90.
- A 2.5 s timer steps through high-score pages, re-setting three labels each time with a new color and gradient.
- Attract also arms a timer that starts a **self-playing demo game**.

### End-of-ball bonus

- `Background/scroll.mp4` plays at priority 89 as a backdrop.
- A script queue writes the bonus lines into eight labels on page 1.
- The count-up is done in **VBScript**: timers at 60–65 ms re-send the caption ten times. PUP's own count-up animation ([chapter 12](../12-label-animations.md#at4-pulse-and-number-count-up)) isn't used.
- "TOTAL BONUS =" and its value are two labels sharing one anchor x: one right-aligned, one left-aligned, so caption and value always meet at the same point whatever their lengths.
- Holding both flippers skips the bonus: stop plus SetBG, then clear the labels.

## Alpha effect videos

```vbscript
Sub pDMD_Splash_Alpha(fname)
    if PupDMD_Effects_Enabled = false then exit sub
    pDMD_Effect_Stop                        ' sends FN 61 (not supported, logged)
    PuPlayer.playlistplayex 11, "Splash_Alpha", fname, 0, 1
End Sub

pDMD_Splash_Alpha "Fireworks30.mov"
```

`PupDMD_Effects_Enabled` comes from the table option "Pinup V2 Enabled", which is **off by default**, so with default options no alpha effects play at all. With it on, screen 11 is a ForcePop child covering screen 5. The `.mov` files carry an alpha channel, so the fireworks are drawn over the HUD with the DMD showing through ([chapter 2](../02-pack-files.md#media-formats)). Volume 0 → 1 %, so the effects are effectively silent.

## Scorbit

Scorbit is off by default and forced off outside Windows. When on, it runs two Windows executables that **generate** `QRcode.png` and `QRclaim.png` into PuPOverlays at runtime. That's why the log shows `Image not found, hiding label` at startup for `QRcodeS.png`, `QRcode.png`, `QRcodeB.png`, `QRclaim.png` (and `Card0.png`): image labels created for files the pack doesn't ship. They stay hidden, which is harmless.

## What VPX doesn't support

| Call | Effect in VPX |
|---|---|
| FN 57 `OH` (`pDMDHookLayers`) | logged warning, no effect |
| FN 60 / 61 (shader effects: fog, fire, shake…) | logged, no effect |
| FN 16 (launch the mini-game `.exe`) and `GameUpdate` | not supported. `GameUpdate` returns 0 |

The framework helpers for FN 2, 41, 45, 50 and 52 are defined in the script but never called.

**The mini-game** is on by default: it switches to a page with no labels, turns the overlay off (FN 3), sends FN 16 and pauses screen 5. Since `GameUpdate` returns 0, it ends after a dozen timer ticks with a score of 0. The net effect is a few seconds of frozen, label-less DMD.

## Pack bugs

- **Missing media:** the `Sewer` and `HighScore` playlists don't exist (for Sewer, the previous loop keeps showing), plus about twenty clips named by the script but absent from the pack (kidnap, mega-mystery super jackpot, several kick/save variants, bonus multiplier).
- **A unit bug:** the one-line splash helper multiplies its duration by 1000, but is called with milliseconds, so " SHOOT AGAIN " stays up until another splash clears it.

None of these break the pack. The rest of the presentation (backgrounds, HUD, pages, animations, effects) works as described.

## Lessons

- Grep every playlist and file literal in the script against the pack before shipping. This one has more than twenty dead references.
- When a pack has several options, keep the script's default layout equal to the shipped default option.
