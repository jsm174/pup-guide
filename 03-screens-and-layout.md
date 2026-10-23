# 3. Screens and layout

## `screens.pup`

```
ScreenNum,ScreenDes,PlayList,PlayFile,Loopit,Active,Priority,CustomPos
2,Backglass,,,0,ForceBack,,
5,FullDMD,Background,loop.mp4,0,ForceBack,80,
11,"Splash Alpha",,,0,ForcePop,,"5,0,0,100,100"
```

At least 6 fields per line (see [chapter 2](02-pack-files.md)); the last two columns are optional. The column names are historical. The PuP Pack Editor shows them under different names, given in brackets:

| # | Column (editor name) | Default | Meaning |
|---|---|---|---|
| 0 | `ScreenNum` (Screen) | 0 | The screen number. |
| 1 | `ScreenDes` (Description) | | Free text, only shown in logs. |
| 2 | `PlayList` (Background Playlist) | | Optional playlist started as this screen's background at load. |
| 3 | `PlayFile` (Background Filename) | | File inside that playlist. Empty means the playlist's next file. |
| 4 | `Loopit` (Transparent) | 0 | Read, but **has no effect** in VPX. Use alpha video, a PuPShapes mask or labels for see-through effects. |
| 5 | `Active` (Mode) | | Screen mode, see below. |
| 6 | `Priority` (Volume %) | 100 | **Not a priority.** It's the screen's volume multiplier in percent ([chapter 7](07-volume-ducking-sounds.md)). Empty, 0, negative or not a number means 100. |
| 7 | `CustomPos` | | Placement inside another screen, see below. |

If two lines define the same screen number, the later one replaces the earlier (children stay attached to the replacement).

### The default background (columns 2 and 3)

When `PlayList` is set and exists, the screen starts it as its background at load, as a SetBG play ([chapter 6](06-playback-engine.md#the-background)) with **volume 0**. Volume 0 is clamped to 1, so a background started this way plays at about **1 % volume**. It's effectively silent.

> **Rule.** To get a background with sound, don't use the `screens.pup` background columns. Start it from a `D0` trigger with action `SetBG` and a volume (or an empty Volume column, which means the playlist's volume). `D0` fires once when the pack starts.
>
> ```
> 3,1,,D0,2,Backglass,Backglass.mp4,,20,,,,SetBG,0
> ```

If the background playlist is the special `PuPFrames` playlist, the file becomes the screen's underlay image instead ([chapter 4](04-playlists.md#special-playlists)).

## Screen numbers

Any integer can be a screen number. Only these have a window of their own ([chapter 1](01-concepts-and-setup.md#where-screens-appear-windows)):

| Number | Traditional name | VPX window |
|---|---|---|
| 0 | Topper | Topper |
| 1 | DMD (4:1 slim) | ScoreView, if screen 5 is unusable |
| 2 | Backglass | Backglass |
| 3 | Playfield | not drawn |
| 4 | Music | not drawn (audio only by convention) |
| 5 | FullDMD / Apron | ScoreView |
| 6 | Backglass 4:3/5:4 | Backglass, if screen 2 is unusable |
| 7+ | anything | not drawn unless placed with CustomPos |

Script-created screens (`Init`) and automatically created parents (see CustomPos) follow the same rules.

## Modes

Column 5 holds the mode keyword, compared case-insensitively:

| Keyword(s) | Mode | Drawn | Audio | Notes |
|---|---|---|---|---|
| `Show` | Show | yes | yes | Normal band. Each media start brings it to the front of its band. |
| `ForceON` | ForceOn | yes, **always-on-top** band | yes | Brought to the front at load and on each media start. |
| `ForcePoP` | ForcePop | yes, always-on-top band | yes | Each media start brings it to the front of its band. |
| `ForceBack` | ForceBack | yes | yes | Normal band. Its media never bring it to the front: it stays where it was created. The usual choice for full-window screens. |
| `ForcePopBack` | ForcePopBack | yes | yes | Normal band. Each media start brings it to the front of its band. |
| `MusicOnly`, `ForceHide` | MusicOnly | no | yes | Plays media for its audio only. |
| `Off`, `Disable`, `JavaScript` | Off | no | no | Disabled. Trigger rows for it are skipped, video plays are refused, and its background doesn't play. |
| anything else | Off | no | | Logged as `Invalid screen mode`. |

Bands and the order inside them are explained under [draw order](#draw-order). ForcePop and ForcePopBack ("pop" screens) hide their window when their media ends (or is stopped) with no background to return to: the underlay, static image and video are no longer drawn, but the labels stay up. The next media start, or FN 1, shows the window again. A still shown in custom image mode doesn't count as an end, so it stays. The only other difference of a pop screen is that it never keeps a background copied with `SetBackGround` ([chapter 6](06-playback-engine.md#the-background)).

Hiding a screen is separate from its mode: a hidden screen keeps its mode and keeps playing, until its next media start shows it again (see below).

### Changing modes at runtime

| Call | Effect |
|---|---|
| `PuPlayer.Show n` | Shows a hidden screen again and brings its video to the front of its band. The mode doesn't change. |
| FN 1 | The same, and also turns its label overlay back on (as FN 3 `OT:1`) and brings its labels to the front. |
| `PuPlayer.Hide n`, FN 28 | Hides the screen: nothing of it is drawn, but its media keeps playing (and is heard), and it keeps its mode. The next media start on the screen (a play, a background start or restart, each pass of a loop, a static image) **shows it again**. Hidden screens don't count when deciding whether a root is usable, so a hidden root with nothing else visible lets its window fall back ([chapter 1](01-concepts-and-setup.md#where-screens-appear-windows)). |
| FN 4 `FS:1` | `ForceOn`, video in the always-on-top band, brought to the front. |
| FN 4 `FS:0` | `ForceBack`, video in the normal band, brought to the front of it. |
| FN 18 | `ForceBack`, video in the normal band. Its place in the order doesn't change. |
| FN 12 | `Off`, then stops the screen. Nothing plays on it any more, background included. |
| `SetScreenEx n,…,popup` | popup 0 → `Show`, 1 → `ForcePop`, 2 or more → `MusicOnly`. The position and size arguments are ignored. |

Changing the mode from a drawn mode to `Off`/`MusicOnly` always stops the current media.

> **Rule.** `Hide`/FN 28 only lasts until the screen's next media start, and on a screen with a looping media or background that's the end of the current pass. To keep a screen out of sight while it plays, hide it after each start, or use FN 20 `AM:1,AV:0`, `MusicOnly` or a CustomPos of size 0.

> **Rule.** `SetScreenEx …,1` changes the mode to ForcePop but doesn't move the screen's video into the always-on-top band. Only a screen created as `ForceON`/`ForcePoP`, or FN 4 `FS:1`, puts it there.

## CustomPos: placing a screen inside another

```
<source screen>,<x>,<y>,<width>,<height>[,<flag>]
```

All four numbers are **percentages** (decimals allowed). Examples:

```
"5,0,0,100,100"       full size on top of screen 5's window
"2,25,10,50,40"       a box on the backglass, 25 % from the left, 10 % from the top
"2,0,0,100,100,6"     flag 6: on the 2.0 engine, media drawn into screen 2 (below)
```

Quote the value: it contains commas.

What it does:

1. The screen becomes a **child** of the source screen. It is drawn in the same window as the source, in the rectangle given.
2. If the source screen doesn't exist (yet), VPX creates a default one: mode `ForceBack` (screen 3 `Off`, screen 4 `MusicOnly`). If the real source screen is defined later in the file, it replaces the default and adopts the children.

> **Rule (define parents first).** List a parent screen **before** its children in `screens.pup`. Screens start in creation order ([below](#draw-order)), so a child listed before its parent starts **under** the parent and stays there until the child plays something (or a ForceBack child: forever).
3. If the source is the screen **itself** (`2,10,10,80,80` on screen 2), it stays a root and is just positioned inside its window.

> **Rule (percent of what?).** The percentages are always of the **window's drawable area** (after the padding settings), not of the parent screen's rectangle. A child of a child is still positioned against the whole window. If screen 5 is placed at `2,0,0,50,50` and screen 11 at `5,0,0,100,100`, screen 11 covers the **whole** backglass, not just screen 5's quarter.

> **Rule.** A screen placed inside screen 0, 1, 2, 5 or 6 (directly, or through other placed screens) is drawn only in that screen's window. It never becomes the root of another window, even with children of its own: if screen 5 is placed inside screen 2, the ScoreView window falls back to screen 1, and if screen 1 isn't usable either, PUP leaves ScoreView to the table's own DMD. A placed screen whose chain ends at any other screen (3, 4, 7…) can still be a window's root when it has children.

Flags and extras:

| Form | Effect in VPX |
|---|---|
| 5 values | Position as described. |
| 6th value `1`, `2`, `3` | Rotation 90/180/270 requested. **Not supported**: logged, the screen is placed unrotated. |
| 6th value `6` | **On the 2.0 engine:** the screen is **hosted** by its parent (below). On the 1.5 engine: ignored. |
| 6th value anything else (`4`, `5`, `7`…) | Ignored. |
| 7 or more values (aspect ratio pair) | **Not supported**: logged. The position from the first 5 values is still used. |
| Source `99` or higher (`99` overlay, `100` absolute pixels, `101` self-relative, …) | **Not supported**: logged, and the whole CustomPos is ignored. The screen is then not a child of anything. |
| Fewer than 5 values | Logged and ignored. |

### Hosted screens (flag 6, 2.0 engine)

On the 2.0 engine, a screen whose CustomPos has `6` as its 6th value (rounded, so `6.0` works) doesn't draw anything of its own. Its media (current media or background video) is drawn **into its parent screen**:

- stretched over the **parent's whole rectangle**: the CustomPos position and size are not used for drawing,
- in the parent's video layer, right after the parent's own video or static image, so it's below the parent's labels and moves with the parent in the draw order,
- at the hosted screen's own opacity (FN 20) and fade-in (FN 55), unless the hosted screen is `Off` or `MusicOnly`. Hiding the hosted screen doesn't hide its media; hiding the parent does.

The hosted screen's underlay, static image and labels are not drawn. An `@` media on a hosted screen that has no labels hides the **parent's** labels ([chapter 10](10-labels.md#hiding-labels-during-a-media)).

Two screens placed in each other (directly, or through other screens) would form a loop. The link that would close it is logged and refused: the screen keeps its earlier place, without that parent.

FN 15 (`{"mt":301,"SN":n,"FN":15,"CP":"…"}`) changes a screen's rectangle at runtime. It does **not** change which screen it belongs to: the parent stays the one from load time, whatever source number the new string has.

Labels on a positioned screen use its rectangle. A label at `xpos 50` is centered in the child, and font sizes are a percentage of the child's height.

## Draw order

Each screen has two layers:

- its **video**: underlay, video (current media or background) or static image,
- its **labels**: every label, pBackground and its PuPOverlays/PuPAlphas image included. A screen gets this layer when it creates its first label.

PUP keeps one stacking order of all these layers, split in two **bands**. In each window, the normal band is drawn first, then the always-on-top band, each from back to front, then the authored DMD overlays from `displays.pup`. Only the window's root screen and the screens placed inside it are drawn. `Off`, `MusicOnly` and hidden screens draw nothing.

| Layer | Always-on-top when |
|---|---|
| video | the screen is `ForceON` or `ForcePoP`, or after FN 4 `FS:1` |
| labels | always, unless FN 23 `FS:0` or FN 33 moved them to the normal band |

So every screen's labels are drawn above every normal-band video, whatever screen they belong to.

What moves a layer:

| Event | Effect |
|---|---|
| Pack load | Videos start in creation order. Then, in file order, each `ForceON` screen, and each screen except `ForceBack` whose default background started, has its video brought to the front, and labels created during the load come to the front. |
| A screen created later (by the script, or replacing one) | Its video starts at the front of its band. |
| First label of a screen (`LabelInit`, `LabelNew`, or a first PuPOverlays/PuPAlphas play) | Its labels come to the front. |
| A media starts on a screen that isn't `Off` or `MusicOnly`: a play, a background start or restart, each pass of a loop, a static image | A hidden screen (`Hide`, FN 28) is shown again. Its video comes to the front, unless the screen was **created** as `ForceBack` or FN 2 `FF:0` turned this off. Then its label overlay is turned on (undoing FN 3 `OT:0`) and its labels come to the front, unless FN 2 `FO:0` or FN 33 turned this off or FN 34 / an `@` media is hiding them. |
| The end of media that ends an FN 34 wait | Its labels come to the front (unless FN 2 `FO:0`). |
| `PuPlayer.Show n` | Its video comes to the front. |
| FN 1, FN 6 | Its video, then its labels, come to the front. FN 1 also shows the screen and turns its label overlay on. |
| FN 3 `OT:1` | Its labels come to the front. |
| FN 4, FN 23 | The video (FN 4) or the labels (FN 23) change band and come to the front of it. |
| FN 5 | Its video goes to the **back**. |
| FN 33 | Both layers move to the normal band and to the back, and go back there on every later media start. |
| FN 57 `OH:1` (2.0 engine) | The screen's labels are drawn right after its video, at the video's place in the order and band, instead of in their own layer. `OH:0` puts them back. |

"Front" and "back" are always within the layer's band.

Consequences worth knowing:

- **Labels over child videos.** A parent's labels, pBackground frame included, are drawn above the videos of the screens placed in it, as long as those are in the normal band (`Show`, `ForceBack`, `ForcePopBack`). This is the classic "background video under the HUD" layout: screen 11 in the [King of the Hill case study](case-studies/king-of-the-hill.md).
- **Splash over everything.** A `ForcePoP` screen's video comes to the front of the always-on-top band each time it starts, so it covers every label that was already there. Its own labels come right after it. A label layer that comes to the front later (another screen starts a media, or a screen's first label) is drawn above the splash.
- **The last one wins.** Between screens of the same band, the one that started a media last is on top. A screen created as `ForceBack` never moves itself, so a screen placed over it and playing anything is drawn above it. Changing a screen to ForceBack later (FN 4 `FS:0`, FN 18) doesn't change this: only the mode it was created with counts.

Within one screen, the layers are:

1. **underlay** image (PuPFrames), stretched to the screen,
2. **video** (current media or background), or the **static image** when no video is playing,
3. **labels** in label order, then every label that is currently animating, on top.

Screen alpha (FN 20) multiplies all three. Overlay alpha (FN 22) multiplies only the labels.

## Clipping

Labels are clipped to their screen's rectangle, rotated ones included.
