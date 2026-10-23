# 13. The PuPDMD framework

Most PuP-enabled original tables include the **PuPDMD framework**, a block of VBScript helper subs (`pDMDLabelSet`, `pDMDSplashPage`, `pupCreateLabel`…) that wrap the raw API. The framework is ordinary table script. This chapter lists what each helper sends and what that actually does, using the rules of the previous chapters.

Framework variables you'll see:

```vbscript
Const pDMD = 5               ' the FullDMD screen (some tables use 1)
Dim pFrameSizeX: pFrameSizeX = 1920
Dim pFrameSizeY: pFrameSizeY = 1080
Dim pUseFramePos: pUseFramePos = 1
```

With `pUseFramePos = 1`, the helpers take positions and sizes in **pixels of a 1920×1080 design frame** and convert them to percentages (`x / 1920 × 100`, `y / 1080 × 100`, truncated to 2 decimals) before sending. The design frame is fixed: a label at x 960 is at 50 % whatever the real window size is.

## Startup

```vbscript
Sub PuPInit
    Set PuPlayer = CreateObject("PinUpPlayer.PinDisplay")
    PuPlayer.B2SInit "", pGameName      ' full load of pupvideos/<pGameName>
    PuPlayer.LabelInit pDMD             ' create the overlay + pBackground
    pSetPageLayouts                     ' LabelNew / pupCreateLabel calls
    pDMDSetPage(pDMDBlank)              ' page 0, made the default page
    pDMDStartUP
End Sub
```

Call `PuPInit` at the end of `Table1_Init`.

## Creating labels

| Helper | Sends | Notes |
|---|---|---|
| `pupCreateLabel name, text, font, size, color, x, y, page, vis` | `LabelNew pDMD, name, font, size, color, 0, 1, 1, 1, 1, page, vis` then `LabelSet pDMD, name, text, vis, "{'mt':2,'xpos':x,'ypos':y,'fonth':size,'v2':1}"` | Center/center aligned. `v2` makes x/y literal (0 = left/top edge). `size` is in design pixels → `fonth` cell height %. |
| `pupCreateLabelImage name, file, x, y, w, h, page, vis` | `LabelNew pDMD, name, "", 50, …, 0, 1, 1, 0, 1, page, vis` then `LabelSet pDMD, name, file, vis, "{'mt':2,'width':w,'height':h,'xpos':x,'ypos':y,'v2':1}"` | An image label, centered on (x, y), sized w×h. |

`LabelNew` sets both the label's visibility and its default visibility from `vis`, so the follow-up `LabelSet` (whose visibility only applies on the current page) doesn't change anything here.

> **Note.** Some copies of the framework have a stray line inside `pupCreateLabel`, starting `float size, LONG color, float angle, …`. It's not VBScript, so it's a syntax error. Delete it if you copy the framework into a table.

## Text and visibility

| Helper | Sends | What it does |
|---|---|---|
| `pDMDLabelSet name, text` | `LabelSet pDMD, name, text, 1, ""` | Sets the text. Shows the label **only if its page is current**. |
| `pDMDLabelShow name` / `pDMDLabelHide name` | `LabelSet …, "`u`", 1/0, ""` | Shows/hides on any page. Not while animating. |
| `pDMDLabelVisible name, v` | `LabelSet …, "`u`", v, ""` | Same. |
| `pDMDSetHUD v` | `pDMDLabelVisible "pBackGround", v` | Shows/hides the pBackground HUD frame only. |
| `pAllVisible v` | FN 3 `OT:v` on screen 5 | Shows/hides **all** labels of screen 5. |
| `pDMDLabelDispose name` | `{'mt':2,'dispose':1}` | Deletes the label. |

## Properties

All of these send `LabelSet pDMD, name, "`u`", 1, "{'mt':2,…}"`. Note the `1`: **every property helper also makes the label visible** (on any page, since the caption is `` `u` ``).

| Helper | Properties sent | Notes |
|---|---|---|
| `pDMDLabelSetPos name, x, y` | `xpos`, `ypos` | Design pixels. |
| `pDMDLabelSetSizeImage name, w, h` | `width`, `height` | Design pixels. |
| `pDMDLabelSetSizeText name, h` | `size` | `size` is a **point size**, larger than `fonth` for the same number ([chapter 11](11-label-properties.md#position-and-size)). |
| `pDMDLabelSetAutoSize name, w, h` | `autow`, `autoh` | |
| `PDMDLabelSetAlign name, xa, ya` | `xalign`, `yalign` | |
| `pDMDLabelSetRotate(Text) name, a` | `rotate` | Tenths of a degree. |
| `pDMDLabelSetZoom name, z` | `zoom` | |
| `pDMDLabelSetColor name, c` | `color` | |
| `pDMDLabelSetAlpha name, a` | `alpha` | |
| `pDMDLabelSetColorGradient name, c1, c2` | `color`, `gradstate` 1 (0 if equal), `gradcolor` | Full-height gradient. |
| `pDMDLabelSetColorGradientPercent name, c1, c2, p` | `color`, `gradstate` p, `gradcolor` | Top p % solid. |
| `pDMDLabelSetGrayScale name, g` | `grayscale` | |
| `pDMDLabelSetFilter name, f` | `filter` | |
| `pDMDLabelSetShadow name, c, x, y, on` | `shadowcolor`, `shadowtype` 1/0, `xoffset`, `yoffset` | Offsets are **percent of the font height**. |
| `pDMDLabelSetBorder name, c, x, y, on` | `shadowcolor`, `shadowtype` 2/0, `xoffset`, `yoffset` | Outline. |
| `pDMDLabelSetOutShadow name, c, x, y, outline, on` | `shadowcolor`, `shadowstate`, `xoffset`, `yoffset`, `outline` | `shadowstate` is applied before `outline` ([chapter 11](11-label-properties.md)): with `on` 1 and `outline` 1 the result is an outline. |
| `pDMDLabelSendToBack name` | `zback:1` | Just above pBackground, not below it. |
| `pDMDLabelSendToFront name` | `ztop:1` | |
| `pDMDLabelStopAnis name` | `stopani:1` | |

## Animations

| Helper | Animation sent | Notes |
|---|---|---|
| `pDMDLabelFlash name, t, c` | `at:1`, `fq:150`, `len:t` | `t` below 20 is taken as seconds (× 1000). Visible afterwards. |
| `pDMDLabelFlashFilter name, t, f` | `at:9`, `fq:150`, `fm:f` | `t` below 20 is seconds. |
| `pDMDLabelPulseText name, text, ms, c` | `at:4`, 80→120 %, `pspeed:0` | Visible afterwards. |
| `pDMDLabelPulseTextEX …, vis, zs, ze` | `at:4`, `zs`→`ze` | |
| `pDMDLabelPulseNumber name, text, ms, c, n1, n2, fmt` | `at:4` + `numstart/numend/numformat` | Counts over `len` (no `mlen`), so the count lasts the whole animation. |
| `pDMDLabelPulseImage name, ms, vis` | `at:4` on the image | |
| `pDMDLabelWiggleText name, text, ms, c` | `at:8`, −45…45°, `rspeed:5` | |
| `pDMDLabelWiggleImage name, ms, vis` | `at:8` | |
| `pDMDLabelMoveHorz name, text, ms, c, s, e` | `at:2`, `xps:s`, `xpe:e`, `tt:2` | Visible afterwards (`vis` 1). |
| `pDMDLabelMoveVert …` | `at:2`, `yps`, `ype`, `tt:2` | |
| `pDMDLabelMoveTO name, text, ms, c, x1, y1, x2, y2` | `at:2`, all four, `tt:2` | Converts design pixels only when `x1+y1+x2+y2 > 4`, so off-screen codes (−1/0/1) stay codes. |
| `pDMDLabelMoveHorzFade` / `VertFade` | same + `af:700`, `vis` 0 | Fades during the last 700 ms and is **hidden** afterwards. |
| `pDMDLabelMoveTOFade` | as MoveTO but `tt:6`, + `af:700`, `vis` 0 | |
| `pDMDScrollBig name, text, t, c` | `at:2`, `xps:1`, `xpe:-1`, `mlen:len`, `tt:0`, `vis` 0 | Right-to-left scroll, hidden afterwards. `t` below 20 is seconds. |
| `pDMDScrollBigV …` | `at:2`, `yps:1`, `ype:-1`, `mlen:0.8×len` | Bottom-to-top. `t` below 20 is seconds. |
| `pDMDZoomBig name, text, t, c, vis, z1, z2` | `at:3`, `ns:1`, `mlen:0.4×len` | `ns` → z1/z2 are percent of the label's own size. `t` below 20 is seconds. |
| `pDMDLabelFadeOut name, ms` | `at:5`, 255→0, `vis` 0 | Hidden afterwards. |
| `pDMDLabelFadeIn name, ms` | `at:5`, 0→255, `vis` 1 | |
| `pDMDLabelFadePulse name, ms, c` | `at:6`, 70↔255, `pspeed:40` | |
| `pDMDScreenFadeOut name, ms` | `at:7`, 255→0, `vis` 0 | **The whole overlay stays at alpha 0** afterwards ([chapter 12](12-label-animations.md#at7-overlay-fade)). |
| `pDMDScreenFadeIn name, ms` | `at:7`, 0→255, `vis` 1 | Restores the overlay. |
| `pDMDLabelClone name, text, ms, c, x, y` | `at:10`, `dl:1000`, `xp`, `yp`, `ad:1`, `vis` 0 | **Position mismatch:** with `pUseFramePos` the helper converts x/y to **percent**, but `xp`/`yp` are **pixels**. A clone meant for (960, 540) lands at (50, 50) pixels, near the top-left corner. Pass pixels of the real screen, or set `pUseFramePos` 0 for these calls. |
| `pDMDLabelCloneDelay …, dl` | same with `dl` | Same mismatch. |

## Animated images

| Helper | Properties | Notes |
|---|---|---|
| `pDMDPNGAnimate name, ms` | `animate:ms` | 0 stops. Makes the label visible. |
| `pDMDPNGAnimateEx name, s, e, loop` | `gifstart`, `gifend`, `gifloop` | `s = e` shows that single frame. |
| `pDMDPNGShowFrame name, f` | `gifstart:f`, `gifend:f` | Single frame. |
| `pDMDPNGAnimateOnce name, ms` | `animate`, `gifloop:0`, `aniendhide:1` | Plays once, then hides. |
| `pDMDPNGAnimateReset name` | `animate:0`, `gifloop:1`, `aniendhide:0`, `gifstart:-1` | |
| `pDMDPNGAnimateOnceAndDispose name, ms` | + `anidispose:1` | Image released at the end. |

## Pages

| Helper | Sends | What it does |
|---|---|---|
| `pDMDSetPage p` | `LabelShowPage pDMD, p, 0, ""` | Switch to page p and make it the **default**. |
| `pDMDSplashPage p, t` | `LabelShowPage pDMD, p, t, ""` | Show p for `t` (**< 501 = seconds, ≥ 501 = ms**), then back to the default. |
| `PDMDSplashPagePlaying p` | `LabelShowPage pDMD, p, 500, "hidehudplay"` | Hide pBackground, show p, return when the current video **ends**. Fallback timeout: **500 seconds**, and that timeout (or any page change before the video ends) returns the page but leaves **pBackground hidden** ([chapter 10](10-labels.md#labelshowpage)). |
| `PDMDSplashPagePlayingHUD p` | `LabelShowPage pDMD, p, 500, "returnplay"` | Same, but keep pBackground. |
| `pHideOverlayDuringCurrentPlay` | FN 34 on screen 5 | Hide all labels and ignore label calls until the current video ends. |

> **Rule.** Start the video **before** calling `PDMDSplashPagePlaying`, `PDMDSplashPagePlayingHUD` or `pHideOverlayDuringCurrentPlay`. They return at the next end of media of any kind. If the video was refused and only a looping background plays, that's the end of the background's current pass; if nothing plays at all, the splash page stays for 500 s and the FN 34 hide until something plays and ends.

```vbscript
PuPlayer.playlistplayex pDMD, "RandomScoring", "Fire Missile 1.mp4", 0, 1
PDMDSplashPagePlaying 4
```

## Media

| Helper | Sends | What it does |
|---|---|---|
| `pDMDPause` / `pDMDResume` / `pDMDStop` | `playpause` / `playresume` / `playstop` | |
| `pDMDVolumeDef v` | `setVolume pDMD, v` | Default volume for later plays. |
| `pDMDVolumeCurrent v` | `setVolumeCurrent pDMD, v` | Volume of what's playing. |
| `pDMDSetLoop v` | `SetLoop pDMD, v` | |
| `pDMDBackground v` | `SetBackGround pDMD, v` | |
| `pDMDSetBackFrame file` | `playlistplayex pDMD, "PUPFrames", file, 0, 1` | Sets the underlay (under the video). Needs `PUPFrames` in `playlists.pup`. |
| `pDMDBackLoopStart list, file` | `playlistplayex pDMD, list, file, 0, 1` + `SetBackGround pDMD, 1` | Plays the file at 1 % volume and copies it into the background slot: it plays once, then loops as the background. **The second call copies whatever is current**: if the first was refused (priority 1 while something of priority ≥ 2 plays), *that* media becomes the background. |
| `pDMDBackLoopStop` | `SetBackGround pDMD, 0` + `playstop pDMD` | Clears the background slot, then stops: the screen is left empty. |
| `AudioDuckPuP id, v` | FN 42 `DV:v` | Auto-restores when screen `id`'s media ends. |
| `AudioDuckPuPAll id, v` | FN 42 `DV:v`, `ALL:1` | **Never** auto-restores ([chapter 7](07-volume-ducking-sounds.md#ending-a-duck)). |
| `PuPEvent n` | `B2SData "D" & n, 1` | Fires trigger rows `D<n>`. |

## Helpers with no effect in VPX

These send custom functions VPX doesn't support ([chapter 9](09-sendmsg-custom-functions.md#accepted-but-not-supported)), so they do nothing: `pSetAspectRatio` (FN 50), `pDisableLoopRefresh` (FN 2), `pSafeLoopModeCurrentVideo` (FN 41), `pSetLowQualityPc` (FN 45), `pDMDSetTextQuality` (FN 52), `pSetVideoPosMS` (FN 51), `PUPDisplayAsJukebox` / `PuPlayListPrior` / `PuPlayListNext` (FN 30/31), `pDMDHookLayers` (FN 57), shader effect helpers (FN 60/61).

`pDMDAlwaysPAD` (FN 46) **is** supported: `~` becomes two spaces and text captions are padded with a space on each side.
