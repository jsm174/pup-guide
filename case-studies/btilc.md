# Big Trouble in Little China

A fairly elaborate pack for an original table (no ROM), built for the **PinUP 2.0 engine**. It uses most of what the guide describes: a DMD screen positioned on the backglass, a hosted screen for alpha-channel effects, a HUD built from image labels, pages, moving score labels and script-driven media. This page walks through release 1.0.1 end to end.

| | |
|---|---|
| Table | `BTILC_2026_Original_1.0.1.vpx` |
| Pack folder | `pupvideos/BTILC` (an empty `v1.0.1.txt` marks the release) |
| Engine | **PinUP 2.0**: `usePuPDriver.ini` names `PinUPDisplayX2.exe` |
| Started by | `PuPlayer.B2SInit "", "BTILC"` from `PuPInit`, called by `Table1_Init` |
| Installed option | "2 Screen DMD on BG no Topper": the root `.pup` files are identical to that option folder, and `ScreenType.txt` = 2 |
| Backglass | screen 2, with screens 5 and 11 inside it |
| ScoreView | none: screen 5 is inside 2 and there's no screen 1 |
| Topper | unused |
| Audio only | screen 4 (music), never played: the table plays its music through VPX |

## The files

```
BTILC_2026_Original_1.0.1.vpx
pupvideos/
  BTILC/
    screens.pup  playlists.pup  triggers.pup  usePuPDriver.ini  ScreenType.txt  v1.0.1.txt
    FONTS/          albertusnovablack.ttf  "Galpon Black.ttf"
    PuPOverlays/    DefaultDMD.png … DefaultDMD6.png  DefaultDMDA.png  EB1.png … none.png  L0.png … R9.png …
    Splash_Alpha/   Fireworks30.mov  LopanEffect15.mov  RainEffect30.mov  SewerEffect15.mov
    Background/  Backglass/  BallSave/  Jackpots/  Misc/  Mode1/ … Mode6/  Wizard/ …
    EndOfBall/  MiniGame/  Tilt/ (empty)  Topper/
    PuP-Pack_Options/   (eight option folders for the Windows .bat installers)
    *.bat  PuPDesktopPunch.exe  sQRCode.exe  sToken.exe  sToken.dat
```

In VPX, the `.bat`/`.exe` files never run ([chapter 2](../02-pack-files.md#things-vpx-ignores)). `ScreenType.txt` is read by the **table script**, not by VPX. `usePuPDriver.ini` is read by VPX to choose the engine.

## The engine

```ini
#DO NOT REMOVE THIS FILE KEEP IN ROOT OF PuPPACK to Change Default PUP driver.
[PINUP]
Driver=PinUPDisplayX2.exe
```

The value is compared case-insensitively, so `PinUPDisplayX2.exe` selects the **2.0 engine**, and the log says `PinUP engine: 2.0 (PinUpDisplayX2)` ([chapter 1](../01-concepts-and-setup.md#the-engine-pinup-15-or-20)). The root file is the same as `PuP-Pack_Options/Pinup-2.0`. Copying `PuP-Pack_Options/Pinup-1.5/usePuPDriver.ini` (`Driver=PinUPDisplayX.exe`) over it selects 1.5. This page describes the 2.0 behaviour and notes where 1.5 would look different.

Of the 2.0 engine's differences, this pack only meets three: CustomPos flag 6, FN 57 and the text y-scale. The rest don't come into play:

| 2.0 difference | Here |
|---|---|
| Custom image mode on by default | The script never plays a still image as media. Images only appear as label captions. |
| `@` media, FN 34 | No file has `@` in its name, and the FN 34 helper is never called. |
| Hidden labels on a hidden or idle pop screen | All labels are on screen 5, which is ForceBack. |
| Shadow types 3 and 4, `shadowq` | Only types 1 and 2 are sent. |
| FN 55, 56, 58, 62 | Not sent. |
| Sound effects | The pack has no `PUPSOUNDS`. |

## `screens.pup`

```
ScreenNum,ScreenDes,PlayList,PlayFile,Loopit,Active,Priority,CustomPos
2,Backglass,,,0,ForceBack,,
4,Music,,,0,MusicOnly,,
5,FullDMD,,,0,ForceBack,,"2,0,0,100,100"
11,"Splash Alpha",,,0,ForcePop,,"5,0,0,100,100,6"
```

What VPX makes of it:

| Screen | Window | Role |
|---|---|---|
| 2 ForceBack | **Backglass** root | Never plays anything in this option. It's only the root that screen 5 sits in. |
| 4 MusicOnly | none | Unused. |
| 5 ForceBack, `2,0,0,100,100` | inside the Backglass, full size | The FullDMD. Carries all the labels (`pDMD = 5`). Its video covers screen 2 entirely. |
| 11 ForcePop, `5,0,0,100,100,6` | **hosted** by screen 5 | Alpha-channel effect videos. Flag 6 on the 2.0 engine: screen 11 draws nothing of its own, and its media is drawn into screen 5's video layer, stretched over screen 5's rectangle ([chapter 3](../03-screens-and-layout.md#hosted-screens-flag-6-20-engine)). |

**VPX windows.** Screen 5 is placed inside screen 2, so it's drawn on the backglass only and can't be the ScoreView root, even though it has a child. ScoreView falls back to screen 1, which doesn't exist, so PUP draws nothing there and leaves the window to other renderers ([chapter 1](../01-concepts-and-setup.md#where-screens-appear-windows)). The whole DMD, HUD included, is on the backglass.

**Draw order.** At startup the script sends FN 57 `OH:1` to screen 5 (`pDMDHookLayers`). On the 2.0 engine, screen 5's labels are then drawn right after its video, in its place in the normal band ([chapter 3](../03-screens-and-layout.md#draw-order)). The backglass, back to front:

```
screen 2 (nothing)  →  screen 5 video  →  screen 11 effects (hosted)  →  screen 5 labels (HUD, scores)
```

This order never changes. Screen 2 and 5 are both ForceBack, so neither moves, and screen 5 was created after 2. Nothing else is drawn in the window, so nothing can come between screen 5's video and its labels. Effects always play **under** the HUD.

**On the 1.5 engine** both flag 6 and FN 57 are ignored. Screen 11 becomes an ordinary ForcePop child over all of screen 5, and screen 5's labels go back to their own layer in the always-on-top band:

```
screen 5 video  →  screen 5 labels  ⇄  screen 11 effects
```

The last two swap: each effect that starts covers the HUD, and screen 5's next media start (a new clip, or a pass of its background loop) brings the labels back above it.

**Text size.** Screen 5's rectangle is the whole backglass window, so on the 2.0 engine its text heights follow the backglass window's shape ([chapter 10](../10-labels.md#text-height-on-the-20-engine)). The HUD art is 1920 × 1080: on a 16:9 backglass the factor is 1 and nothing changes, on a 4:3 backglass every text label is drawn at 75 % of its size. The 1.5 engine never scales.

The volume column is empty everywhere, so every screen has a multiplier of 100.

## The option variants

The pack ships eight option folders for the Windows installers. `playlists.pup` is identical in all of them. In VPX ([chapter 3](../03-screens-and-layout.md#screen-numbers)):

| Installer (`ScreenType.txt`) | Difference from the installed set | In VPX |
|---|---|---|
| 2 Screen DMD on BG no Topper (2) | identical (the folder also has a `playlists-old.pup`, which is never read) | as above |
| 3 Screen - SlimDMD (2), folder "3 Screen - BG and Slim DMD" | adds a ForceOn screen 1; `D0` rows also start `Backglass.mp4` on 2 and `Topper.mp4` on 1 | the backglass looks the same, since screen 5 covers screen 2's loop (which is still heard). ScoreView falls back to screen 1 and shows the `Topper.mp4` loop |
| 3 Screen - FullDMD (3) | screen 5 is its own root; adds a ForcePopBack screen 13 inside 5; `D0` also starts `Backglass.mp4` on 2 | Backglass shows `Backglass.mp4`, ScoreView shows the DMD. Screen 13 never plays anything. If it did, on the 2.0 engine its video would cover screen 5's labels, since FN 57 put them with screen 5's video |
| 4 Screen - BG DMD Topper Separate (4) | as FullDMD, plus a ForceOn screen 0 with a `D0` `Topper.mp4` loop | as FullDMD, and the Topper window gets the loop |
| 1 - Desktop (0), folder "Desktop - 3 Screen" | same `.pup` files as FullDMD, plus a `pupinit.bat` that starts `PuPDesktopPunch.exe` | as FullDMD; `pupinit.bat` is ignored |
| 5 VR - 3 Screen (5) | as FullDMD, but screen 2 is ForcePopBack, and a MiniGame screen 12 inside 5 is listed **before** its parent | screen 12 is unused by the script |
| 6 / 7 - Use Pinup-v1.5 / v2.0 | only `usePuPDriver.ini` | selects the 1.5 or 2.0 engine |

The script reads `ScreenType.txt`, but only on Windows:

```vbscript
DMDType = CInt(ObjFile.ReadLine)
if PlatformOS = "windows" Then
    bStandAlone = False
Else
    bStandAlone = True
    DMDType = 3
End If
```

Every type maps to the same DMD screen, 5. On Windows, type 2 sends the mini-game to screen 2 ([below](#what-vpx-doesnt-support)) and moves the sewer timer labels at some values. Elsewhere the type is always 3, the mini-game is turned off and so is Scorbit.

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
0,Skillshot,,0,-1,100,0
0,Splash_Alpha,,0,-1,100,0
0,Tilt,,0,-1,100,0
0,Topper,,0,-1,100,0
0,Wizard,,0,-1,100,0
```

- `ScreenNum` 0 is ignored. Every playlist exists on every screen ([chapter 4](../04-playlists.md#each-screen-has-its-own-copy)).
- `AlphaSort` 0 → alphabetical. `RestSeconds` −1 → no rest. Videos are always named explicitly anyway.
- `PuPOverlays` is listed twice. The second row logs `Duplicate playlist` and is skipped, which is harmless.
- `FONTS` and `PuP-Pack_Options` are listed as playlists. Nothing ever plays from them. `Tilt` is an empty folder: kept with a warning, and never played from (the tilt clip is in `Misc`). Harmless too.
- `EndOfBall` and `MiniGame` aren't listed. Nothing plays from them: the bonus backdrop is played from `Background`, and `MiniGame` holds the Windows mini-game program.
- Listing `PuPOverlays` makes it the **special overlay playlist** ([chapter 4](../04-playlists.md#special-playlists)). This pack doesn't play from it, though: it uses its images as **label captions**, where the first path component (`PuPOverlays`) is resolved as a playlist.

## `triggers.pup`

```
ID,Active,Descript,Trigger,ScreenNum,PlayList,PlayFile,Volume,Priority,Length,Counter,RestSeconds,Loop,Defaults
1,0,***STARTUP***,D0,11,Background,Background2.mp4,,20,,,,SetBG,0
3,1,,D0,5,Background,Background2.mp4,,1,,,,SetBG,0
```

- Row 1 is inactive (`Active` 0), so it's never loaded ([chapter 5](../05-triggers.md#columns)).
- Row 3 fires on the startup event `D0`: screen 5 gets `Background2.mp4` as its **background loop**. The Volume column is empty, so it plays at the playlist volume (100) and its soundtrack is heard.
- Screen 2 gets nothing: its `D0` row only exists in the other options.

Everything else is driven from the script. The `PuPEvent` helper (`B2SData "D"&n`) is never called.

The `D0` loop doesn't last. Three seconds after the table starts, attract mode returns screen 5 to rest, and the script's resting loop for "no mode" is a different file, at volume 0 (1 %):

```vbscript
Select Case Mode(CurrentPlayer,0)
    Case 0
        PuPlayer.playevent pDMD,"Background","Background.mp4",0,20,6,0,""
```

So `Background2.mp4` plays only for the first seconds, with sound. From then on the base loop is `Background.mp4`, practically silent.

## Starting it from the script

```vbscript
Sub PuPInit
Set PuPlayer = CreateObject("PinUpPlayer.PinDisplay")
PuPlayer.B2SInit "", pGameName                       ' pGameName = "BTILC"
...
    pupPackScreenFile = PuPlayer.GetRoot & "BTILC\ScreenType.txt"   ' GetRoot = ".../pupvideos/"
    ...
    CheckPupVersion                                  ' Windows only: looks for PinUPDisplayX.exe
    PuPlayer.LabelInit pDMD                          ' pDMD = 5: overlay + pBackground
    pSetPageLayouts
    pDMDSetPage(pDMDBlank)                           ' page 0
    ...
    pDMDStartUP                                      ' page 1, then FN 57 OH:1 on screen 5
End Sub
```

There's no ROM, so nothing loads automatically. `B2SInit` performs the full load and `D0` starts screen 5's background. `CheckPupVersion` returns at once outside Windows; on Windows it looks for `PinUPDisplayX.exe` next to the `pupvideos` folder and only prints the version it finds. One second after start the script shows the attract frame, and after three seconds attract mode starts.

## The HUD: fonts and labels

```vbscript
Const dmddef = "Galpon Black"
Const dmdNum = "Albertus Nova Black"
```

Both fonts are shipped in `FONTS` and found by their family names ([chapter 10](../10-labels.md#fonts)). With a Windows-only font name here, every text label would be blank.

```vbscript
Sub pSetPageLayouts
    pDMDAlwaysPAD        ' FN 46: pad text with spaces, '~' = line break between spaces
    ...
    pupCreateLabelImageDMD "DMDOverlay","PupOverlays\\DefaultDMD.png",0,0,100,100,1,0
    pupCreateLabelImageDMD "DMDGracieOverlay","PupOverlays\\DefaultDMD_GRACIE.png",0,0,100,100,1,0
    pupCreateLabelImageDMD "DMDMMOverlay","PupOverlays\\DefaultDMD_MIDNIGHT.png",0,0,100,100,1,0
    pupCreateLabelImageDMD "DMDOverlayAttract","PupOverlays\\DefaultDMDA.png",0,0,100,100,5,0
    pupCreateLabelImageDMD "QC","PupOverlays\\QuestComplete.png",0,0,100,100,1,0
    pupCreateLabelImageDMD "EB","PupOverlays\\EB1.png",0,0,100,100,1,0
    ...
    PuPlayer.LabelNew pDMD, "PointsL", dmddef, FontSizeSplash+2, cGold, 0, 1, 1, 5, -20, pScores, 0
    ...
    PuPlayer.LabelNew pDMD, "Splash", dmddef, FontSizeSplash+4, cGold, 0, 1, 1, 50, 32, pScores, 0
    ...
End Sub

Sub pupCreateLabelImageDMD(lName, lFilename,xpos, ypos, Iwidth, Iheight, pagenum, lvis)
    PuPlayer.LabelNew pDMD,lName ,"",50,RGB(100,100,100),0,1,1,1,1,pagenum,lvis
    PuPlayer.LabelSet pDMD,lName,lFilename,lvis,"{'mt':2,'width':"&IWidth&",'height':"&Iheight&",'xpos':"&xpos&",'ypos':"&ypos&"}"
end Sub
```

What's going on:

- **Image HUD frames as labels.** Each frame is an image label with an empty font (fine for images), sized `width/height` 100 × 100 % of screen 5. There are about 85 labels in all, every one on screen 5.
- **Position 0 = centered.** The frames are placed at `xpos 0, ypos 0` *without* `v2`. Because they're center-aligned, position 0 means 50 % ([chapter 10](../10-labels.md#positioning)), so each frame is **centered** and covers the whole screen. With `v2` they'd be centered on the top-left corner instead.
- **`"PupOverlays\\DefaultDMD.png"`**: VBScript doesn't escape backslashes, so the caption really contains `\\`. Label captions collapse repeated separators and accept `\` on every OS. `PupOverlays` matches the `PuPOverlays` playlist case-insensitively.
- **Pages.** The gameplay frames are on page 1 (`pScores`), the match labels on page 2, the attract frame and texts on page 5, all created hidden (`lvis` 0). Pages 0 and 1 don't reset visibility on a switch, but pages 2 and 5 do ([chapter 10](../10-labels.md#labelshowpage)).
- **Off-screen start positions.** Labels like `PointsL` are created at `ypos -20`, above the screen, so they're out of view until an animation moves them in.
- **A frame per state.** `UpdateEddieSaves` swaps the main frame between `DefaultDMD.png` and `defaultDMD1.png` … `defaultDMD6.png` as the player earns saves. The file names differ in case from the files on disk, which doesn't matter.

### Label styling

- **Gradients:** about 45 calls send `color`, `gradstate:1` and `gradcolor`, a full-height vertical gradient ([chapter 11](../11-label-properties.md#color-and-look)).
- **Borders:** over 60 calls set an outline (`shadowtype` 2), the same on both engines ([chapter 11](../11-label-properties.md#shadow-and-outline)). These helpers send a `` `u` `` caption with visible 1, which applies even off-page; harmless here because the captions are empty, and empty labels aren't drawn ([chapter 10](../10-labels.md#pages)).
- **Z-order:** `zback:1` is re-sent on every main-frame swap, keeping the frame just above pBackground and under the text. The Gracie, Midnight Madness and Quest Complete frames are shown with `ztop:1` instead, so they're drawn **above** the text labels (animating labels still go on top) ([chapter 11](../11-label-properties.md#order)).
- Only move-with-fade and pulse animations are actually called; the zoom, wiggle, flash and clone helpers are defined but unused.

## Moving score labels

```vbscript
PuPlayer.LabelSet pDMD,"PointsL",FormatScore(PointsAdjusted),0,"{'mt':1,'at':2, 'len':1000, 'fc':" & cWhite & ",'xps':5,'xpe' :35,'yps':20,'ype' :78, 'tt':6 ,'ad':1, 'af':700}"
```

- Sent for every score of 50,000 or more.
- `visible` 0 → the label returns to **hidden** when the animation ends ([chapter 12](../12-label-animations.md#rules-common-to-every-animation)).
- `at:2` moves the label's top-left corner from (5 %, 20 %) to (35 %, 78 %), with cubic in-out easing (`tt:6`), over 1 s.
- `af:700` fades it out over the last 700 ms.
- `ad:1` is ignored: a new score replaces the running animation.

## Image labels used as switches

```vbscript
PuPlayer.LabelSet pDMD, "EB", "PupOverlays\\EB2.png",1,""     ' show 2 extra balls
PuPlayer.LabelSet pDMD, "EB", "PupOverlays\\none.png",1,""    ' show nothing
```

The pack ships a transparent `none.png` and swaps it in to "clear" the label. That's more reliable than an empty caption, which is **ignored** on image labels ([chapter 10](../10-labels.md#caption-rules)). The table's `pDMDHideAnimate` helper (`LabelSet …, "", 0, ""`) still works for hiding: the empty caption is ignored, but the `visible` 0 applies (when the label is on the current page).

The match sequence works the same way with digit images: `"PuPOverlays\\L"& MatchPre &".png"` cycles `L0.png` … `L9.png`, and `R0.png` … `R9.png` on the right.

## Media from the script

Every play goes to screen 5, except the effects on screen 11:

```vbscript
PuPlayer.playevent pDMD,"BallSave","BS_Gracie.mp4",nPupVideoVolume,55,0,0,""     ' Normal, pri 55
PuPlayer.playevent pDMD,"Misc","Extraball.mp4",nPupVideoVolume,87,0,0,""         ' finds Misc/ExtraBall.mp4
PuPlayer.playevent pDMD,"Misc","Tilted.mp4",nPupVideoVolume,89,0,3,""            ' cut after 3 s
PuPlayer.playevent pDMD,"Background","Background.mp4",0,20,6,0,""                ' SetBG
```

- `nPupVideoVolume` is 125, the jackpots use 125 too, and the base loop 0 (1 %).
- The priorities make important videos interrupt lesser ones and refuse lower ones ([chapter 6](../06-playback-engine.md#priorities)). A ball-save video (55) can't cut off an extra-ball video (87 or 90).
- `Tilted.mp4` with `seconds` 3: the tilt video ends 3 s after the request, and the background comes back ([chapter 8](../08-script-api.md#playevent)). If a video above 89 were playing, the tilt request would be refused and the playing video left alone.

The script also plays `HighScore.mp4`, `EnterInitials.mp4` (type 8, SkipSamePrty) and `ComboRecord.mp4` (type 11, QueueSameP) from a `HighScore` playlist, but this pack has no `HighScore` folder or `playlists.pup` row, so those calls only log `Playlist not found`. With the folder present, `ComboRecord.mp4` (priority 92) would wait behind `EnterInitials.mp4` (same priority) instead of replacing it.

### Modes: "stop, then SetBG the mode loop"

- At mode start, the script sets the **mode loop** as SetBG at priority 20 and volume 125 (audible, unlike the base loop at volume 0), and 200 ms later plays the intro clip at priority 65. The loop starts at once, dropping whatever was playing, then the intro interrupts it after 200 ms; the loop comes back when the intro ends ([chapter 6](../06-playback-engine.md#the-background)). If a clip above priority 20 is still playing at mode start, the SetBG is refused and the old background stays. The script re-sends the mode loop at 3.3 s and 8.1 s: refused while the intro (65) is still playing, a volume-only no-op once the loop is already playing, and the fix-up when the first one was refused.
- After any interruption, `CheckBackgroundVideo` stops the screen 50 ms later and sets the current resting loop as SetBG 100 ms after that (20 call sites):

  ```vbscript
  Sub CheckBackgroundVideo (caller)
      GeneralPupQueue.Add "PuPlayer.playstop pdmd","PuPlayer.playstop pdmd",94,50,0,0,0,False
      GeneralPupQueue.Add "PlayModeLoopVideo","PlayModeLoopVideo",20,150,0,0,0,False
  End Sub
  ```

  The stop clears the current media, so the SetBG at 20 isn't refused unless something new started in those 100 ms. `PlayModeLoopVideo` picks the loop for the current state: sewer, Midnight Madness, mystery, super spinners, Wildman, multiball, a mini-wizard, the current mode, or `Background.mp4`. It's the pack's single way back to rest, which makes it hard to end up on the wrong loop.
- Mode progress clips use **SkipSamePrty** at 62, so rapid shots don't cut each other off ([chapter 6](../06-playback-engine.md#skipsameprty)).
- Loops are broken with StopFile: the four skill-shot value clips loop at 60 while the ball is in the plunger lane and are StopFiled at 61; the game-over loop (95) is StopFiled when attract starts.

> **Lesson.** Keep one routine that returns a screen to its current resting loop, and call it after anything that interrupts.

### Attract mode

- Attract starts by StopFiling the game-over loop (followed by `CheckBackgroundVideo`), switching to page 5 and **then** showing the attract frame. The order matters: switching to a page above 1 resets its labels to their created visibility ([chapter 10](../10-labels.md#labelshowpage)).
- After 500 ms, `Credits/CreditIntro.mp4` plays at priority 90.
- A 2.5 s timer steps through the high-score pages, re-setting three labels each time with a new color and gradient.
- With the "Auto Play" option (on by default), the same timer starts a **self-playing demo game**.

### End-of-ball bonus

- `Background/scroll.mp4` plays at priority 90 as a backdrop, right away or 2.9 s after the ball-lost clip (89).
- A script queue writes the bonus lines into labels on page 1.
- The count-up is done in **VBScript**: a 65 ms timer re-sends the captions step by step. PUP's own count-up animation ([chapter 12](../12-label-animations.md#at4-pulse-and-number-count-up)) isn't used.
- "TOTAL BONUS =" and its value are two labels sharing one anchor x: one right-aligned, one left-aligned, so caption and value always meet at the same point whatever their lengths.
- Holding both flippers skips the bonus: `CheckBackgroundVideo`, then clear the labels.

## Alpha effect videos

```vbscript
Sub pDMD_Splash_Alpha(fname)
   if PupDMD_Effects_Enabled = false then exit sub
	pDMD_Effect_Stop                          ' sends FN 61 (not supported, logged)
    PuPlayer.playlistplayex 11,"Splash_Alpha",fname,0,1
End Sub

pDMD_Splash_Alpha "Fireworks30.mov"
```

`PupDMD_Effects_Enabled` comes from the table option "Pinup V2 Enabled", which is **off by default**, so with default options no alpha effects play at all. With it on, the `.mov` files (QuickTime Animation, with an alpha channel) play on screen 11 ([chapter 2](../02-pack-files.md#media-formats)). On the 2.0 engine screen 11 is hosted, so they're drawn over screen 5's video and **under** the HUD, stretched to the whole DMD (see [`screens.pup`](#screenspup)). On the 1.5 engine they cover the HUD until screen 5's next media start brings the labels back above them. Volume 0 → 1 %, so the effects are effectively silent.

The same option also gates the shader effects (FN 60, a "shake" on tilt, lightning in some modes), which VPX doesn't support.

## Scorbit

Scorbit is off unless the script is edited (`ScorbitActive = 0`), and forced off outside Windows. When on, it runs two Windows executables that **generate** `QRcode.png` and `QRclaim.png` into PuPOverlays at runtime. The script creates the QR labels whether or not Scorbit is on, so the log shows `Image not found, hiding label` at startup for `QRcodeS.png`, `QRcode.png`, `QRcodeB.png` and `QRclaim.png`, along with `Card0.png`, `L00.png` and `R00.png`: image labels created for files the pack doesn't ship. They stay hidden, which is harmless; the match labels get real digit images when the match starts.

## What VPX doesn't support

| Call | Effect in VPX |
|---|---|
| FN 57 `OH:1` (`pDMDHookLayers`) | Supported on the 2.0 engine: screen 5's labels are drawn with its video ([chapter 9](../09-sendmsg-custom-functions.md#supported)). Ignored on 1.5 |
| FN 60 / 61 (shader effects: shake, lightning…) | logged as unknown, no effect ([chapter 9](../09-sendmsg-custom-functions.md#accepted-but-not-supported)) |
| FN 16 (launch the mini-game `.exe`) and `GameUpdate` | not supported. `GameUpdate` returns 0 ([chapter 8](../08-script-api.md#overview)) |

The framework helpers for FN 2, 3, 30, 31, 34, 41, 42, 45, 50, 51 and 52 are defined in the script but never called. FN 3 is only sent directly, by the mini-game.

**The mini-game** (option "Mini Game", on by default) only exists on Windows, where it can be a mystery award. It switches to page 77, which has no labels, sends FN 16 and starts polling `GameUpdate`. Since that returns 0, it ends after a dozen timer ticks with a score of 0. With the installed layout (type 2), the FN 3 and pause go to screen 2, which has no labels and plays nothing, so the DMD video keeps running without its HUD for a few seconds. With the other layouts they go to screen 5, and the DMD freezes, label-less, for the same time.

## Pack bugs

- **Missing playlists:** `HighScore` (three calls) and `Sewer`: the sewer resting loop is requested as `"Sewer","SewerLoop.mp4"`, but the file is in `Misc`, where two other calls find it. After `CheckBackgroundVideo`, the stop restarts the previous resting loop and that loop stays.
- **Missing files:** about twenty clips named by the script aren't in the pack:
  - `Kidnap`, `Sewer` and `MiaoYin` variants of the kickback, save, bonus-multiplier and playfield-multiplier clips. The pack spells the last one `MiaoYen` (`KICK_MiaoYen.mp4`, `BM_MiaoYen.mp4`…), so the MiaoYin mode never gets them.
  - `BallSave/BS_Kidnap.mp4`, `Jackpots/MegaMystSuperJackpot.mp4` (the pack has `MegaMysterySuperJackpot.mp4`), `Kickbacks/BonusMultiplier.mp4` and `Misc/BonusMultiplier.mp4` (the file is in `Multipliers`).
  - `BallLost/BL_12.mp4`: the drain clip is picked at random from 1 to 26, and number 12 is missing, so about one drain in 26 shows no clip.
  - `Splash_Alpha/Fireworks30_6sec.mov`, the effect for one mode (only with "Pinup V2 Enabled").
  - Dead code that would fail too: `Skillshot/Skillshot.mp4`, `GameOver/GAMEOVER1.mp4` and `Misc/GauntletLoop.mp4` are named in routines that are never called or never reached.

  A missing file is logged and refused, and whatever was playing keeps playing.
- **A unit bug:** the one-line splash helper multiplies its duration by 1000, but most callers pass milliseconds, so " SHOOT AGAIN ", " TILT WARNING! ", " INSERT COIN " and the credit count stay up until something else clears the label (the tilt warning has its own clear 2.25 s later).

None of these break the pack. The rest of the presentation (backgrounds, HUD, pages, animations, effects) works as described.

## Lessons

- Grep every playlist and file literal in the script against the pack before shipping, including names built from variables (`"BL_" & i & ".mp4"`) and the spelling of each character's name. This one has more than twenty dead references.
- A DMD positioned on the backglass takes the ScoreView with it. If the cabinet has a DMD screen, pick an option where screen 5 is its own root.
- On the 2.0 engine, FN 57 plus a hosted effects screen gives a fixed order (video, effects, HUD) that no media start can upset. On 1.5 the same layout makes the effects and the HUD trade places.
