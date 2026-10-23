# 1. Concepts and setup

## The mental model

A PinUP pack is a folder of media plus three small CSV files that tell the player:

1. **which screens exist** and how they behave (`screens.pup`),
2. **which subfolders are playlists** (`playlists.pup`),
3. **what to play when something happens on the machine** (`triggers.pup`).

At runtime, two things drive the screens:

- **Trigger events.** Switches, lamps, solenoids, GI, mechs, segment displays and recognized DMD frames coming from PinMAME or B2S, plus events the table script sends with `B2SData`. Each event is matched against `triggers.pup`, and a matching row starts, stops or changes media on one screen.
- **Script calls.** The table creates a `PinUpPlayer.PinDisplay` object and calls methods on it: play this file, show this page of text, animate this label, send this JSON message.

Each screen runs its own small media engine. It has a **current media** with a priority, a **queue** of media waiting behind it, a **background** loop to fall back to, an **underlay image**, an optional **mask**, and a **label overlay** with pages of text and images drawn on top. The rest of this guide is mostly the exact rules of that engine.

```
                  ┌──────────────── one PUP screen ────────────────┐
 trigger / script │  label overlay (pages, pBackground, animations) │  ← top
 ───────────────► │  current media  ◄── queue ◄── background loop   │
                  │  static image (when nothing is playing)          │
                  │  underlay image (PuPFrames)                      │  ← bottom
                  └─────────────────────────────────────────────────┘
```

## How VPX finds a pack

A pack is a folder whose name is the **game key**, which is the PinMAME ROM name (`mytable_l1`) or, for original tables, whatever name the script passes to `B2SInit`. VPX looks for that folder in this order and uses the first that exists (case-insensitively):

1. `<table folder>/pupvideos/<namespace>/<game>`, e.g. `pupvideos/pinmame/mytable_l1`
2. `<table folder>/pupvideos/<game>`
3. `<PUPFolder setting>/pupvideos/<namespace>/<game>`
4. `<PUPFolder setting>/pupvideos/<game>`

The `<namespace>` levels only apply when the game comes from a controller (PinMAME reports `pinmame`). A pack next to the table always wins over the global folder.

> **Rule.** The setting `PUPFolder` points to the folder that **contains** `pupvideos`, not to `pupvideos` itself. With no global folder configured, only per-table `pupvideos` folders are used, which is the normal setup.

## How a pack starts

There are three ways a pack gets loaded. They don't do the same thing.

### A. Automatically, from PinMAME

When a controller such as PinMAME starts a game and a pack folder exists for its ROM, VPX loads the whole pack. The table script doesn't need to do anything. If several controllers expose the same game, a `pinmame` one is preferred.

If the script has already started PUP itself (case B) before this happens, the automatic load is skipped and the log says:

```
PinUpPlayer was instantiated and initialized from the table script before table init; skipping automated PuP loading.
```

### B. From the script, with `B2SInit`

```vbscript
Set PuPlayer = CreateObject("PinUpPlayer.PinDisplay")
PuPlayer.B2SInit "", "BTILC"      ' first argument is ignored
```

This performs a **full load** of the folder named by the second argument. Original tables (no ROM) and the PuPDMD framework use this.

### C. From the script, with `Init` only

```vbscript
PuPlayer.Init 2, "mySample"
PuPlayer.playlistadd 2, "Ramps", 1, 30
```

This is the old "scripted" style. `Init` remembers the pack folder (loading its `FONTS` and `PUPSOUNDS`) and creates screen 2 as a plain `Show` screen. It does **not** read `screens.pup` or `playlists.pup`, and it does not start the trigger event stream. You add playlists with `playlistadd` and play from the script.

> **Rule (the Init trap).** If a script calls `Init` for a game and then calls `B2SInit` (or PinMAME starts) for the **same** game, VPX sees "same ROM, already set up" and keeps the scripted state. `screens.pup` and `playlists.pup` are **never loaded** in that case. Use one style or the other.

> **Rule.** Calling `Init` for a screen that already exists **replaces** it with a default `Show` screen. That discards the mode, CustomPos, volume, labels and playlists that `screens.pup` gave it. Never call `Init` after `B2SInit`.

### What a full load does, in order

1. Unloads any previously loaded pack.
2. Locates the folder (above). If none is found, nothing else happens. Fonts from `FONTS` and sounds from `PUPSOUNDS` are loaded as soon as the folder is found.
3. Reads `displays.pup` (authored DMD overlays, [chapter 14](14-dmd-overlays.md)) and registers the in-game editor for it.
4. Reads `playlists.pup`.
5. Reloads the fonts.
6. Reads `screens.pup` line by line. Each screen copies every playlist and loads the `triggers.pup` rows for its own number. A screen with a default playlist starts it as its background.
7. Starts the event stream (PinMAME/B2S state polling and DMD frame matching).
8. Sends the startup event **`D0` = 1**. A trigger on `D0` fires at pack start, which is how packs start their initial backgrounds.

Calling `B2SInit` again for the same game, as many tables do, only makes sure the event stream is running and sends `D0` again if it wasn't running. Calling it with a **different** game unloads everything and loads the new pack.

### When a pack unloads

- when the game ends (the table stops),
- `PuPlayer.CloseApp` or `PuPlayer.PuPClose`,
- `B2SInit` with a different game.

Destroying the `PinDisplay` script object does **not** unload the pack. Tables often create several of them.

## Where screens appear: windows

VPX has three ancillary windows a pack can draw into. Each window shows one **root screen**:

| VPX window | Root screen | Fallback if unusable |
|---|---|---|
| Topper | 0 | none |
| Backglass | 2 | 6 |
| ScoreView (the DMD/"FullDMD" window) | 5 | 1 |

Any other screen number (3, 4, 7, 11, 13, …) is only drawn if it is placed **inside** one of those root screens with a CustomPos ([chapter 3](03-screens-and-layout.md)). Screens 3 (Playfield), 4 (Music), 7–10 and so on are never drawn on their own. Their audio still plays.

A root is **usable** when its screen tree (the root plus everything placed inside it) contains at least one screen that is not `Off`, `MusicOnly` or hidden (`Hide`/FN 28). An `Off` or hidden root with a visible child is still usable: the root itself isn't drawn, its children are. A root that is itself placed inside another screen is usable only if it has children. When screen 5 is unusable, ScoreView falls back to screen 1, and Backglass falls back from 2 to 6 the same way. When PUP has nothing usable for a window, it doesn't claim the window, so another renderer such as B2S can draw there.

The windows themselves (position, monitor, enabled or not) are set up in VPX's display settings, not by the pack.

> **Rule.** PUP draws its screens only when VPX's FFmpeg libraries are available. Without them, nothing is drawn in a window that has a usable root screen, labels and `displays.pup` overlays included. `displays.pup` overlays only still render in windows without a usable root.

Media housekeeping (noticing that a video ended, advancing queues, restarting backgrounds, page timers, volume fades) runs on its own timer, so it also works for screens that are never drawn, such as a MusicOnly screen or a pack whose windows are disabled.

## Settings

All PUP settings are in the `[Plugin.PUP]` section of the VPX ini and in the in-game settings UI.

| Key | Default | Meaning |
|---|---|---|
| `Enable` | `1` | Turns the PUP plugin on or off. |
| `PUPFolder` | empty | Folder **containing** a global `pupvideos` folder. |
| `MainVol` | `1.0` | Master volume for all PUP media, 0.0 to 1.0. Changes apply live. Sound effects from `PUPSOUNDS` are not scaled by it. |
| `BGPadLeft`, `BGPadRight`, `BGPadTop`, `BGPadBottom` | `0` | Pixels of padding inside the Backglass window. The root screen is drawn in the remaining area. |
| `SVPadLeft` … `SVPadBottom` | `0` | Same for the ScoreView window. |
| `TopperPadLeft` … `TopperPadBottom` | `0` | Same for the Topper window. |

Example:

```ini
[Plugin.PUP]
Enable = 1
MainVol = 0.8
SVPadTop = 40
SVPadBottom = 40
```

## Audio routing

All PUP media audio and sound effects are mixed into one audio source named **"PinUp Player"**, which VPX sends to its **backglass** audio output.

## Time

- Video playback is clocked by the **game time** when VPX provides it, so videos stay in sync with the table.
- If the game clock stops advancing for more than 100 ms (pause, debugger, a long script call), videos keep playing on the wall clock instead of freezing.
- A video's timeline starts at its **first rendered frame**, not when it was requested. If the window stops rendering for more than 250 ms (table loading, a hitch), the video waits instead of skipping ahead. A video on a screen that isn't drawn within a second of the request (a MusicOnly or hidden screen) runs on its own clock instead.
- Label page timers, static image lengths and label animations run on the wall clock. Page timers and image lengths are checked by the media housekeeping timer, whether or not the screen is drawn.
- Trigger actions run on the main thread one frame after the event (a 1 ms dispatch). They are never instantaneous relative to the script.
