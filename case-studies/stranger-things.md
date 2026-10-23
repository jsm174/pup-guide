# Stranger Things

An original table whose pack has **no `.pup` files at all**. The script creates every screen, adds every playlist and plays every clip. It's the reference example of a script-only pack, and of priorities used as layers.

| | |
|---|---|
| Pack folder | `pupvideos/STLE`, media folders only |
| Started by | `PuPlayer.Init` for screens 2, 4, 7, 8 and 0, at **module level** (before `Table1_Init`) |
| Backglass | screen 2 |
| Topper | screen 0 (when the table's topper option is on) |
| ScoreView | not PUP: no screen 5 or 1, so the window is left to the table's own DMD |
| Audio only | screens 4 (music), 7 (mode music), 8 (callouts) |

An identical copy of the pack also sits at the table root as `STLE/`. VPX never reads it: packs are found under `pupvideos` ([chapter 1](../01-concepts-and-setup.md#how-vpx-finds-a-pack)).

## Starting a script-only pack

Everything runs at module level, before the table initializes:

```vb
PuPlayer.Init pBackglass,"STLE"
PuPlayer.Init pMusic,"STLE"
PuPlayer.Init pAudio,"STLE"
PuPlayer.Init pCallouts,"STLE"

PuPlayer.SetScreenex pBackglass,0,0,0,0,0
PuPlayer.SetScreenex pMusic,0,0,0,0,2
PuPlayer.hide pMusic
' … same for pAudio and pCallouts

PuPlayer.playlistadd pMusic,"audiobg", 1 , 0
PuPlayer.playlistadd pBackglass,"scene", 1 , 0
' … 43 playlistadd calls in all

PuPlayer.playlistplayex pBackglass,"scene","base.mov",0,1
PuPlayer.SetBackground pBackglass,1
```

`SetBackground pBackglass,1` copies the clip that was just started into the background slot, so it loops whenever nothing else plays ([chapter 6](../06-playback-engine.md#the-background)).

Then `Table1_Init` calls `LoadEM`, which starts a B2S controller for the game `STLE`. VPX sees a controller for the game already bound by `Init` and **keeps the pack as it is** instead of reloading it. That's what makes a script-only pack survive: a reload would look for a `screens.pup`, find none, and drop the script's screens ([chapter 1](../01-concepts-and-setup.md#c-from-the-script-with-init-only)). Keep the `Init` folder name equal to the controller's game name.

`Hide` on a MusicOnly screen is redundant: it's never drawn anyway ([chapter 3](../03-screens-and-layout.md#modes)).

## Priorities as layers

| Priority | Content |
|---|---|
| 1 | Normal clips |
| 2 | Mode loops (levitate, pool, radio, compass) |
| 3 | Multiball loops (party, upside-down, Barb, Bad Men, wizard) |
| 4 | Tilt |
| 7 | High-score entry loop |

The background `scene/base.mov` plays at volume 0 (1 %) whenever nothing else does.

### The `clear.mov` stub

To end a loop, the script plays `videobarb/clear.mov`, an **85 ms** clip, at the **same priority** as the loop, then turns looping off with `SetLoop 2,0`. Equal priority replaces ([chapter 6](../06-playback-engine.md#priorities)), the stub ends almost at once, and the background restarts ([chapter 6](../06-playback-engine.md#the-background)). At priority 1, the same stub works as a flipper "skip this scene".

It's a tidy idiom: a stop that respects priorities, unlike `playstop`, which stops anything.

### Music around modes

The soundtrack loops on screen 4. A mode pauses it (`playpause 4`), loops its own track on screen 7, then ends that with a silent `clear.mp3` and resumes screen 4 (`playresume 4`).

Callouts duck the music by hand: FN 11 lowers screens 2, 4 and 7, and a 2.2 s timer restores them. The `-NN` file-name ducking or FN 42 would do the same automatically ([chapter 7](../07-volume-ducking-sounds.md#ducking)).

## The HUD

- About 135 labels on screen 2: scores and ball in a right column, mode counters on the left, a timer-driven rules ticker, high-score tables.
- **Fonts with double spaces.** Two family names contain double spaces (`Fundamental  Brigade`, `Fundamental 3D  Brigade`). The script uses the same double spaces, and VPX collapses repeated spaces on both sides, so they match either way.
- **A layered pseudo-3D title.** A black label in the 3D-extruded font sits under a white label in the flat font, at the same position. A timer steps `'size'` from 0 to 6 and fades the color in.
- `'size'` below 1 is treated as 1, so the first two title frames look the same.
- `size` is in points, about 1.5× a `LabelNew` size ([chapter 11](../11-label-properties.md#position-and-size)). Labels created at 2 jump in size when first set with `'size':2`; the title labels compensate (created at 9, set to 6).
- `"avantgarde"` matches no shipped font and logs "Font not found", but the labels using it are never shown.

## What doesn't work (in VPX or anywhere)

Several "end" clips can't play, because of how priorities work ([chapter 6](../06-playback-engine.md#priorities)), not because of VPX:

- **Lower priority while a loop plays is refused.** `levitateend.mov` at 1 against the `levitate` loop at 2; `compassend` at 1 against a 64 s loop at 2; `barbendshort` and `badmenend` at 1 against loops at 3. The timers that try again later re-request a loop that's already playing, which doesn't restart it.
- **Replaced at once.** `radioend.mov` at 2 is accepted, and the very next statement plays `clear.mov` at 2, which replaces it immediately.
- The one that works: `willend.mov` plays at 3, equal to its loop.

## Lessons

- A script-only pack needs no `.pup` files at all: `Init`, `SetScreenEx`, `playlistadd` and a SetBG loop are enough.
- An "end" clip must use a priority **at least equal** to the loop it ends, and nothing at that priority may follow it in the same tick.
- A near-zero-length clip at the loop's priority is a priority-respecting stop.
- Copy font family names exactly, and don't name fonts you don't ship.
