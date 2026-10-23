# 9. SendMSG and custom functions

## Message format

`PuPlayer.SendMSG` takes a JSON object. Keys and string values can use double quotes (doubled inside a VBScript string) or single quotes:

```vbscript
PuPlayer.SendMSG "{ ""mt"":301, ""SN"":5, ""FN"":42, ""DV"":30 }"
PuPlayer.SendMSG "{'mt':301, 'SN':5, 'FN':42, 'DV':30}"
```

The `mt` field selects the message type:

| `mt` | Meaning | Fields |
|---|---|---|
| 301 | Custom function on a screen | `SN` screen (**99 = every screen** 0–19 that exists), `FN` function, plus the function's fields |
| 402 | Play a PUPSOUNDS sound | `ns` name (with `.wav`), `sX`, `sY`, `sZ`, `vol` (default 100), `sp` |
| 400, 401 | Sound add / play | accepted and **ignored** |
| missing | | logged as `No message type found` |
| anything else | | logged as not implemented |

> **Rule.** Only `mt` 301 and 402 do something. The other numbered message types that some documentation describes (label, playlist and screen commands as JSON) aren't accepted through `SendMSG` in VPX. Use the matching script methods.

`mt` 301 without `SN` is logged and ignored. With an `SN` whose screen doesn't exist: logged, ignored. Without `FN`: ignored silently.

The same custom function JSON (without `mt`/`SN`) is what a `CustomFunc` trigger row carries in its PlayList column ([chapter 5](05-triggers.md#customfunc)). There it applies to the row's screen.

Missing numeric fields read as the value shown in the Default column of each function below.

## Custom functions (`FN`)

### Supported

| FN | Fields (default) | Effect |
|---|---|---|
| 1 | | **Show** a hidden screen. The mode doesn't change. |
| 3 | `OT` (0) | Label overlay **on/off** for the screen: `OT` ≠ 0 shows all labels, 0 hides them all (including pBackground and PuPOverlays images). Video is unaffected. |
| 4 | `FS` | **Stay on top**: `FS` 1 → ForceOn mode, topmost tier. Anything else → ForceBack, normal tier. |
| 5 | | Send the screen to the **back** of the draw order. |
| 6 | | Bring the screen to the **front** of the draw order (within its tier: it doesn't become topmost). |
| 10 | `VL` (0) | Set the **default volume** of **every** screen. |
| 11 | `VL` (0) | Set this screen's **current** volume. |
| 12 | | Set the screen **Off** and stop it. Nothing plays on it any more, background included. |
| 15 | `CP` | Change the screen's **CustomPos** rectangle (string `"src,x,y,w,h"`). The parent screen doesn't change. |
| 18 | | Back to **ForceBack**, not topmost. |
| 20 | `AM` (0), `AV` (255) | **Screen opacity**: `AM` 1 → `AV`/255 for everything on the screen (underlay, video, labels). `AM` 0 → fully opaque. |
| 22 | `AM` (0), `AV` (255) | **Label overlay opacity**: `AM` 1 → all labels × `AV`/255. `AM` 0 → 1. Video is unaffected. Label animation type 7 also drives this value. |
| 26 | `VL` (0) | Screen **volume multiplier** in percent (like `screens.pup` column 6). Applied immediately. 0 gives the 1 % floor. |
| 27 | `PN` (0), `HS` (0), `EX` ("") | Same as `LabelShowPage screen, PN, HS, EX` ([chapter 10](10-labels.md#pages)). |
| 28 | | **Hide** the screen: not drawn, media keeps playing and is heard. The mode doesn't change. |
| 34 | | **Hide labels until the current media ends**: the whole overlay is hidden, and `LabelSet` and `LabelShowPage` calls on the screen are **ignored**, until the next end of media on this screen (its end, a stop, a queue advance or a loop pass). |
| 40 | `VL` (0) | Same as 11. |
| 42 | `DV` (0), `ALL` (0) | **Duck** other screens ([chapter 7](07-volume-ducking-sounds.md#ducking)). `DV` −1 restores. |
| 46 | `PA` (0) | **Pad mode** for the screen's labels: `PA` 1 → `~` becomes two spaces instead of a line break, and every text caption set afterwards gets a space on each side. |
| 54 | `DV` (−1) | Default duck level for `-name` media files without two digits. |
| 55 | `FS` | Video **fade-in** step 1–255 ([chapter 6](06-playback-engine.md#visual-continuity)). 255, 0 or negative = off. |

> **Rule (FN 34).** FN 34 waits for the next end of media of any kind. If only a looping background plays, that's the end of its current pass; if nothing plays at all, the labels stay hidden until something plays and ends. Send FN 34 right after starting the media it's meant for.

> **Rule (FN 3 vs pDMDSetHUD).** FN 3 hides the **whole** label overlay of a screen. `pDMDSetHUD 0` only hides the pBackground label. They're independent: labels show only when FN 3 is on.

> **Rule (FN 34 vs FN 3).** FN 34 switches the overlay off the same way FN 3 `OT:0` does, and the end of media switches it back **on**. An overlay you hid with FN 3 reappears when an FN 34 wait ends.

### Accepted but not supported

These are logged and have no effect:

| FN | Was meant for |
|---|---|
| 2 | Window "bring to front on play" / overlay-on-play flags (`FF`, `FO`) |
| 16 | Launching an external program (mini games) |
| 17 | Moving an external window |
| 30, 31 | Jukebox mode, next/previous |
| 32 | Font quality |
| 33 | DMD mirroring / keep at back |
| 41 | Seamless loop |
| 45 | Slow-PC mode |
| 50 | Workspace aspect ratio |
| 51 | Seek to a position in ms |
| 52 | Text anti-aliasing level |
| 53 | Label canvas size |
| 57 | Overlay hide toggle (`OH`). Logged as a warning. |

Any other `FN` (7, 8, 9, 13, 14, 19, 21, 23, 24, 25, 29, 35–39, 43, 44, 47, 60, 61, 100+…) is logged as `Unknown function not implemented`. Packs that call shader or effect functions (FN 60/61) just get no effect.

## Recipes

```vbscript
' Hide/show every label on the FullDMD
PuPlayer.SendMSG "{'mt':301,'SN':5,'FN':3,'OT':0}"
PuPlayer.SendMSG "{'mt':301,'SN':5,'FN':3,'OT':1}"

' Make the backglass video 50 % transparent
PuPlayer.SendMSG "{'mt':301,'SN':2,'FN':20,'AM':1,'AV':128}"

' Duck everything while screen 6 plays a callout (auto-restores when it ends)
PuPlayer.playlistplayex 6, "Callouts", "", 100, 50
PuPlayer.SendMSG "{'mt':301,'SN':6,'FN':42,'DV':20}"

' Hide labels for the duration of a full-screen video
PuPlayer.playlistplayex 5, "Wizard", "intro.mp4", 100, 90
PuPlayer.SendMSG "{'mt':301,'SN':5,'FN':34}"

' Move a child screen at runtime
PuPlayer.SendMSG "{'mt':301,'SN':11,'FN':15,'CP':'5,25,25,50,50'}"

' Soft fade-in for new videos on the backglass
PuPlayer.SendMSG "{'mt':301,'SN':2,'FN':55,'FS':12}"

' Same function on every screen
PuPlayer.SendMSG "{'mt':301,'SN':99,'FN':10,'VL':70}"
```
