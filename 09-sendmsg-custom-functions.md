# 9. SendMSG and custom functions

## Message format

`PuPlayer.SendMSG` takes a JSON object. Keys and string values can use double quotes (doubled inside a VBScript string) or single quotes. String values are read without their quotes, and `\n`, `\t`, `\\` and escaped quotes inside them are decoded:

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
| 1 | | **Show** a hidden screen, turn its label overlay on (as FN 3 `OT:1`), and bring its video, then its labels, to the front of their bands ([draw order](03-screens-and-layout.md#draw-order)). The mode doesn't change. |
| 2 | `FF` (0), `FO` | **Bring to front on play**: `FF` 1 → each media start brings the screen's video to the front, anything else (or no `FF`) → it doesn't. `FO` (only if present) 0 → each media start no longer brings its labels to the front or turns its overlay back on, anything else → it does. Both are on by default, except `FF` for screens created as ForceBack. |
| 3 | `OT` (0) | Label overlay **on/off** for the screen: `OT` ≠ 0 shows all labels and brings them to the front, 0 hides them all (including pBackground and PuPOverlays images) until the next media start turns the overlay back on (unless FN 2 `FO:0`). Video is unaffected. |
| 4 | `FS` | **Stay on top**: `FS` 1 → ForceOn mode, video in the always-on-top band. Anything else → ForceBack, normal band. Either way the video comes to the front of its band. |
| 5 | | Send the screen's **video** to the **back** of its band. Its labels don't move. |
| 6 | | Bring the screen's video, then its labels, to the **front** of their bands. It doesn't change band. |
| 7 | `CP` (0) | **Custom image mode**: `CP` 1 → on, anything else → off ([chapter 6](06-playback-engine.md#images-in-a-normal-playlist)). |
| 10 | `VL` (0) | Set the **default volume** of **every** screen. |
| 11 | `VL` (0) | Set this screen's **current** volume. |
| 12 | | Set the screen **Off** and stop it. Nothing plays on it any more, background included. Also turns custom image mode off. |
| 15 | `CP` | Change the screen's **CustomPos** rectangle (string `"src,x,y,w,h"`). The parent screen doesn't change. |
| 18 | | Back to **ForceBack**, video in the normal band. Its place in the order doesn't change. |
| 20 | `AM` (0), `AV` (255) | **Screen opacity**: `AM` 1 → `AV`/255 for everything on the screen (underlay, video, labels). `AM` 0 → fully opaque. |
| 22 | `AM` (0), `AV` (255) | **Label overlay opacity**: `AM` 1 → all labels × `AV`/255. `AM` 0 → 1. Video is unaffected. Label animation type 7 also drives this value. |
| 23 | `FS` | **Labels on top**: `FS` 1 → the screen's labels in the always-on-top band (the default), anything else → normal band, so later videos can cover them. Either way they come to the front of their band. |
| 24 | `EN` (0) | **Safe mode**, 1.5 engine only: `EN` 1 stops the screen and refuses every play on it, `EN` 0 ends it ([chapter 6](06-playback-engine.md#safe-mode-and-hard-stop)). Ignored on the 2.0 engine. |
| 26 | `VL` (0) | Screen **volume multiplier** in percent (like `screens.pup` column 6). Applied immediately. 0 gives the 1 % floor. |
| 27 | `PN` (0), `HS` (0), `EX` ("") | Same as `LabelShowPage screen, PN, HS, EX` ([chapter 10](10-labels.md#pages)). |
| 28 | | **Hide** the screen: not drawn, media keeps playing and is heard. The mode doesn't change. The next media start on the screen shows it again. |
| 29 | `FT` (0), `FI` (100) | **Fade the screen out**, then stop (`FT` 0), pause (1), stop and hide (2) or pause and hide (3). `FI` is the step interval in ms ([chapter 6](06-playback-engine.md#fading-a-screen-out-fn-29)). |
| 30 | `PM` (0) | `PM` ≠ 0: **jukebox mode** on, for good ([chapter 6](06-playback-engine.md#jukebox-mode-fn-30-and-31)). |
| 31 | `PM` (0) | In jukebox mode: `PM` 1 plays the **previous** file of the playlist, any other positive `PM` the **next** one. Ignored otherwise. |
| 33 | | **Keep at back**: the screen's video and labels move to the normal band and to the back, go back there on every media start, and its labels no longer come to the front on play (as FN 2 `FO:0`). The DMD mirroring part of FN 33 isn't supported (logged). |
| 34 | | **Hide labels until the current media ends**: the whole overlay is hidden, and `LabelSet` and `LabelShowPage` calls on the screen are **ignored**, until the next end of media on this screen (its end, a stop, a queue advance or a loop pass), or on the 2.0 engine the start of a media without `@`. A media with `@` in its path does the same by itself ([chapter 10](10-labels.md#hiding-labels-during-a-media)). |
| 35 | `FT` (0), `FP` (0) | **Jump** within what's playing by `FP` % of its length: `FT` 1 forward, 2 backward ([chapter 6](06-playback-engine.md#length-loop-and-pause-from-the-script)). |
| 36 | | **Custom image mode** on (as FN 7 `CP:1`). |
| 38 | `SD` (0), `SC` (0) | **Default shadow** of labels **created afterwards**, on every screen: `SD` 1 → drop shadow, anything else → none, in color `SC`. Labels that already exist don't change ([chapter 11](11-label-properties.md#shadow-and-outline)). |
| 40 | `VL` (0) | Same as 11. |
| 41 | | **Safe loop** for what's playing: a looping media longer than 1 s wraps 1 s before its end ([chapter 6](06-playback-engine.md#loop)). |
| 42 | `DV` (0), `ALL` (0) | **Duck** other screens ([chapter 7](07-volume-ducking-sounds.md#ducking)). `DV` −1 restores. |
| 44 | `PI` (0) | **Label paint interval** in ms, 16–250 (other values ignored). Label animations run at 32/`PI` of their normal speed ([chapter 12](12-label-animations.md#rules-common-to-every-animation)). Needs the screen's labels to exist. |
| 46 | `PA` (0) | **Pad mode** for the screen's labels: `PA` 1 → `~` becomes a line break with a space on each side, and every text caption set afterwards gets a space on each side. |
| 47 | `KS` (0) | Same as 44 with `KS` as the interval, any value above 10. |
| 51 | `SP` (0) | **Seek** what's playing to `SP` ms ([chapter 6](06-playback-engine.md#length-loop-and-pause-from-the-script)). |
| 53 | `XW` (0), `YH` (0), `FR` (0) | **Label canvas size** `XW` × `YH` pixels (`XW` above 10; needs the screen's labels to exist). On the 2.0 engine it replaces the screen's shape in the [text height scaling](10-labels.md#text-height-on-the-20-engine). Labels are always drawn scaled to the screen, as `FR` 1 asks: with another `FR` and a canvas 5 pixels or more off the screen's size, that's logged as not implemented. No visible effect on the 1.5 engine. |
| 54 | `DV` (−1) | Default duck level for `-name` media files without two digits. |
| 55 | `FS` | 2.0 engine only: video **fade-in** step 1–254 ([chapter 6](06-playback-engine.md#visual-continuity)). 255 or 0 = off, negative ignored. |
| 56 | `FH`, `FS` | Same as 55. `FH` is ignored. |
| 57 | `OH` | 2.0 engine only: `OH` 1 → the screen's labels are drawn with its video, at the video's place in the [draw order](03-screens-and-layout.md#draw-order). `OH` 0 → back in their own layer. No `OH`: ignored. Needs the screen's labels to exist. |
| 58 | `YS` | 2.0 engine only: **label text height factor** for labels created or sized afterwards, `YS` above 0.1 ([chapter 10](10-labels.md#text-height-on-the-20-engine)). Needs the screen's labels to exist. |
| 62 | `SF` (100) | 2.0 engine only: **sound effect volume** in percent, for the whole pack ([chapter 7](07-volume-ducking-sounds.md#playing)). |
| 300 | | **Hard stop**: stop what's playing without an end of media; nothing restarts until the next play ([chapter 6](06-playback-engine.md#safe-mode-and-hard-stop)). |
| 301 | | From now on, an FN 29 fade-out that would pause stops instead. |

> **Rule (FN 34).** FN 34 waits for the next end of media of any kind. If only a looping background plays, that's the end of its current pass; if nothing plays at all, the labels stay hidden until something plays and ends. Send FN 34 right after starting the media it's meant for.

> **Rule (FN 3 vs pDMDSetHUD).** FN 3 hides the **whole** label overlay of a screen. `pDMDSetHUD 0` only hides the pBackground label. They're independent: labels show only when FN 3 is on.

> **Rule (FN 3 and FN 28 don't last).** Every media start on the screen (loop passes and background restarts included) shows a screen hidden by FN 28 and turns an overlay switched off by FN 3 back on. Send FN 2 `FO:0` first if the overlay must stay off while media plays.

> **Rule (FN 34 vs FN 3).** FN 34 switches the overlay off the same way FN 3 `OT:0` does, and the end of media switches it back **on**. An overlay you hid with FN 3 reappears when an FN 34 wait ends.

> **Rule (engine-only functions).** On the 1.5 engine FN 55, 56, 57, 58 and 62 are ignored without a log line, and on the 2.0 engine FN 24 is ([chapter 1](01-concepts-and-setup.md#the-engine-pinup-15-or-20)).

### Accepted but not supported

These are logged and have no effect:

| FN | Was meant for |
|---|---|
| 16 | Launching an external program (mini games) |
| 17 | Moving an external window |
| 32 | Font quality |
| 45 | Slow-PC mode |
| 50 | Workspace aspect ratio |
| 52 | Text anti-aliasing level |

FN 33's DMD mirroring and FN 53's unscaled canvas (`FR` other than 1) are logged too, while the rest of those functions works.

Any other `FN` (8, 9, 13, 14, 19, 21, 25, 37, 39, 43, 48, 49, 59, 60, 61, 63 and up, except 300 and 301) is logged as `Unknown function not implemented`. Packs that call shader or effect functions (FN 60/61) just get no effect.

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

' Soft fade-in for new videos on the backglass (2.0 engine)
PuPlayer.SendMSG "{'mt':301,'SN':2,'FN':55,'FS':12}"

' Fade the backglass out over about 2.6 s, then stop and hide it
PuPlayer.SendMSG "{'mt':301,'SN':2,'FN':29,'FT':2}"

' Jukebox: auto-advance through the playlist, then skip ahead
PuPlayer.SendMSG "{'mt':301,'SN':4,'FN':30,'PM':1}"
PuPlayer.SendMSG "{'mt':301,'SN':4,'FN':31,'PM':2}"

' Same function on every screen
PuPlayer.SendMSG "{'mt':301,'SN':99,'FN':10,'VL':70}"
```
