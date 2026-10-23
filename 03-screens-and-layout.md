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
| `Show` | Show | yes | yes | Plain screen. Never reordered. |
| `ForceON` | ForceOn | yes, in the **topmost** tier | yes | Above every non-topmost screen of the window. |
| `ForcePoP` | ForcePop | yes, topmost tier, **pop** layer | yes | Each new video brings it to the front of its tier. |
| `ForceBack` | ForceBack | yes | yes | Plain screen, never reordered. The usual choice for full-window screens. |
| `ForcePopBack` | ForcePopBack | yes, **pop** layer | yes | Each new video sends it to the back of the order. |
| `MusicOnly`, `ForceHide` | MusicOnly | no | yes | Plays media for its audio only. |
| `Off`, `Disable`, `JavaScript` | Off | no | no | Disabled. Trigger rows for it are skipped, video plays are refused, and its background doesn't play. |
| anything else | Off | no | | Logged as `Invalid screen mode`. |

What "pop" and "topmost" mean for drawing is explained under [draw order](#draw-order). A pop screen does not open or close a window. When it has no media, nothing of its video is drawn, but its underlay, static image and labels still are.

Hiding a screen is separate from its mode: a hidden screen keeps its mode and keeps playing (see below).

### Changing modes at runtime

| Call | Effect |
|---|---|
| `PuPlayer.Show n`, FN 1 | Shows a hidden screen again. The mode doesn't change. |
| `PuPlayer.Hide n`, FN 28 | Hides the screen: nothing of it is drawn, but its media keeps playing (and is heard), and it keeps its mode. Hidden screens don't count when deciding whether a root is usable, so a hidden root with nothing else visible lets its window fall back ([chapter 1](01-concepts-and-setup.md#where-screens-appear-windows)). |
| FN 4 `FS:1` | `ForceOn` and topmost. |
| FN 4 `FS:0`, FN 18 | `ForceBack`, not topmost. |
| FN 12 | `Off`, then stops the screen. Nothing plays on it any more, background included. |
| `SetScreenEx n,…,popup` | popup 0 → `Show`, 1 → `ForcePop`, 2 or more → `MusicOnly`. The position and size arguments are ignored. |

Changing the mode from a drawn mode to `Off`/`MusicOnly` always stops the current media.

> **Rule.** `SetScreenEx …,1` changes the mode to ForcePop but doesn't move the screen into the topmost tier. Only a screen created as `ForceON`/`ForcePoP`, or FN 4, is topmost.

## CustomPos: placing a screen inside another

```
<source screen>,<x>,<y>,<width>,<height>[,<flag>]
```

All four numbers are **percentages** (decimals allowed). Examples:

```
"5,0,0,100,100"       full size on top of screen 5's window
"2,25,10,50,40"       a box on the backglass, 25 % from the left, 10 % from the top
"2,0,0,100,100,6"     flag 6: accepted, no effect
```

Quote the value: it contains commas.

What it does:

1. The screen becomes a **child** of the source screen. It is drawn in the same window as the source, in the rectangle given.
2. If the source screen doesn't exist (yet), VPX creates a default one: mode `ForceBack` (screen 3 `Off`, screen 4 `MusicOnly`). If the real source screen is defined later in the file, it replaces the default and adopts the children.

> **Rule (define parents first).** List a parent screen **before** its children in `screens.pup`. Draw order follows creation order ([below](#draw-order)), so a child listed before its parent is drawn **under** the parent within the same tier.
3. If the source is the screen **itself** (`2,10,10,80,80` on screen 2), it stays a root and is just positioned inside its window.

> **Rule (percent of what?).** The percentages are always of the **window's drawable area** (after the padding settings), not of the parent screen's rectangle. A child of a child is still positioned against the whole window. If screen 5 is placed at `2,0,0,50,50` and screen 11 at `5,0,0,100,100`, screen 11 covers the **whole** backglass, not just screen 5's quarter.

> **Rule.** A root screen that has a CustomPos pointing to another screen is drawn inside that screen. If screen 5 is placed inside screen 2 and has no children of its own, the ScoreView window falls back to screen 1. If it does have children, ScoreView still shows screen 5's tree, and screen 5 then appears in both windows.

Flags and extras:

| Form | Effect in VPX |
|---|---|
| 5 values | Position as described. |
| 6th value `1`, `2`, `3` | Rotation 90/180/270 requested. **Not supported**: logged, the screen is placed unrotated. |
| 6th value anything else (`4`–`7`…) | Ignored. |
| 7 or more values (aspect ratio pair) | **Not supported**: logged. The position from the first 5 values is still used. |
| Source `99` or higher (`99` overlay, `100` absolute pixels, `101` self-relative, …) | **Not supported**: logged, and the whole CustomPos is ignored. The screen is then not a child of anything. |
| Fewer than 5 values | Logged and ignored. |

FN 15 (`{"mt":301,"SN":n,"FN":15,"CP":"…"}`) changes a screen's rectangle at runtime. It does **not** change which screen it belongs to: the parent stays the one from load time, whatever source number the new string has.

Labels on a positioned screen use its rectangle. A label at `xpos 50` is centered in the child, and font sizes are a percentage of the child's height.

## Draw order

Within a window, PUP draws the root screen and all its descendants in this order (back to front):

| Step | Screens | What is drawn |
|---|---|---|
| 1 | not topmost, not pop | underlay + video |
| 2 | not topmost, **pop** | underlay + video + labels |
| 3 | not topmost, not pop | labels |
| 4 | **topmost**, not pop | underlay + video |
| 5 | topmost, not pop | labels |
| 6 | topmost, pop | underlay + video + labels |
| 7 | | authored DMD overlays from `displays.pup` |

Topmost screens are ForceON, ForcePoP, and FN 4 `FS:1`. Pop screens are ForcePoP and ForcePopBack. `Off` screens are skipped entirely. MusicOnly screens draw nothing.

Consequences worth knowing:

- A **ForcePopBack** child over a ForceBack parent draws its video above the parent's video but **below the parent's labels** (step 2 vs 3). This is the classic "background video under the HUD" layout: screen 13 in the [BTILC case study](case-studies/btilc.md).
- A **ForcePoP** child draws above everything, labels included (step 6). This is the classic "splash / alpha effect over the whole DMD" layout.
- Inside one step, screens are drawn in **screen order**: the order they were created (file order of `screens.pup`, then script-created screens), changed by ForcePop plays (to the front), ForcePopBack plays (to the back), FN 5 (to the back) and FN 6 (to the front). A replaced screen moves to the end.

Within one screen, the layers are:

1. **underlay** image (PuPFrames), stretched to the screen,
2. **video** (current media or background), or the **static image** when no video is playing,
3. **labels** in label order, then every label that is currently animating, on top.

Screen alpha (FN 20) multiplies all three. Overlay alpha (FN 22) multiplies only the labels.

## Clipping

Labels are clipped to their screen's rectangle, rotated ones included.
