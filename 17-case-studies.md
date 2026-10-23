# 17. Case studies: what real packs do

This chapter looks at fifteen real packs, PinMAME and original tables, and collects what they have in common: how they're driven, how they layer screens, how they get back to a resting state, and the mistakes that keep showing up. Each pack also has its own page, linked below.

## The packs

| Table | Driven by | What it's good for |
|---|---|---|
| [Indiana Jones](case-studies/indiana-jones.md) | DMD captures | Scene and callout layers with one rect, a priority ladder, capture ID collisions |
| [Monster Bash](case-studies/monster-bash.md) | DMD captures | The simplest layout: one screen per window, rest-gated Normal rows |
| [Transformers](case-studies/transformers.md) | DMD captures | Five zones, modes as Loop + StopFile, SetBG behind a Loop |
| [The Flintstones](case-studies/flintstones.md) | DMD captures | A four-layer backglass with a PuPShapes text mask |
| [Cuphead](case-studies/cuphead.md) | Script events → triggers | A score display made of twelve digit screens |
| [Darkest Dungeon](case-studies/darkest-dungeon.md) | Script events → triggers | Transparent clears, a mode whitelist, and most of the common bugs |
| [Die Hard Trilogy](case-studies/die-hard-trilogy.md) | Script events → triggers | Cut-out backglass art, GI-driven lights, a label mini-game |
| [Total Nuclear Annihilation](case-studies/total-nuclear-annihilation.md) | Script events → triggers | A SetBG slideshow, the ForcePopBack/ForcePop pair |
| [Futurama](case-studies/futurama.md) | Script scheduler, equal priorities | The script owns timing; a label text cache vs page defaults |
| [Guardians of the Galaxy](case-studies/guardians-of-the-galaxy.md) | Script scheduler | A bezel image over an under-video layer |
| [The Matrix](case-studies/the-matrix.md) | Script scheduler | Scene text from a mini-language, a playfield video that can't arrive |
| [Batman 66](case-studies/batman-66.md) | Script scheduler | A frame with holes over four movable TV screens |
| [King of the Hill](case-studies/king-of-the-hill.md) | Script `playevent` only, no triggers | One boxed video screen and a priority ladder |
| [Big Trouble in Little China](case-studies/btilc.md) | Script `playevent` | A full walk-through: screens, HUD from image labels, alpha effects, option variants, mode loops |
| [Stranger Things](case-studies/stranger-things.md) | Script only, no `.pup` files | `Init`/`playlistadd` packs, priorities as layers, a stub clip as a stop |

## Four ways to drive a pack

**1. DMD captures.** PinMAME tables whose script never mentions PUP. The pack starts by itself when the ROM starts ([chapter 1](01-concepts-and-setup.md#a-automatically-from-pinmame)), and almost every row is a `D` trigger tied to a `PupCapture` frame ([chapter 5](05-triggers.md#dmd-frame-triggers)). A few switch rows (`W`) cover what the DMD doesn't show, such as a launch button. These packs depend on the ROM's language and settings, and on capture file names.

**2. Script events to triggers.** Original tables whose script sends numbered events (`B2SData "D"&n` or `"E"&n`) and whose presentation lives in `triggers.pup`. The script stays simple. Several packs filter events in their `PuPEvent` routine: during a mode, only a whitelist passes ([Darkest Dungeon](case-studies/darkest-dungeon.md)), or nothing passes while tilted ([Die Hard Trilogy](case-studies/die-hard-trilogy.md)).

**3. A script scheduler.** The script keeps its own queue of scenes, each with a **measured duration**, and decides what plays when. PinUP's own queue actions ([chapter 6](06-playback-engine.md#queue-actions)) are almost unused: one script call in [BTILC](case-studies/btilc.md), whose playlist is missing anyway, and no trigger row in any of these packs. [Futurama](case-studies/futurama.md) goes furthest: every trigger has the same priority, so each new clip simply replaces the last, and the script holds the ball, starts light shows and ducks music for each scene's length. [Batman 66](case-studies/batman-66.md) measured its clip lengths with `ffprobe` and keeps one queue per screen.

**4. Script only.** [Stranger Things](case-studies/stranger-things.md) ships no `.pup` files at all: `Init`, `SetScreenEx`, `playlistadd` and `playlistplayex` do everything. It works because a controller start for the same game keeps the scripted state ([chapter 1](01-concepts-and-setup.md#c-from-the-script-with-init-only)).

None is "the right one". Captures need no script work but can't see anything the DMD doesn't show. Trigger packs keep the presentation editable without touching the script. A script scheduler gives exact sequencing at the cost of keeping durations in sync with the videos.

## Layering recipes

Every multi-layer design in these packs is a consequence of the [draw order](03-screens-and-layout.md#draw-order): within a window, non-pop videos first, then pop screens (video and labels), then non-pop labels, then topmost screens.

| Recipe | How | Seen in |
|---|---|---|
| Frame over video | The frame is the parent's **pBackground** (a `PuPOverlays` play). Labels draw after every non-pop video, so it covers the children's video. | Batman 66, Indiana Jones, Transformers |
| Cut-out art | Backglass art as pBackground, with a **transparent hole** where a child screen sits | Die Hard Trilogy, Cuphead |
| Frame as a static image | A PNG from a normal playlist on a Show screen that plays no video, moved to the front with FN 6. Covers video, but not labels. | Guardians of the Galaxy |
| Video under the HUD | A **ForcePopBack** child: above the parent's video, below its labels | TNA, King of the Hill, BTILC |
| Video over everything | A **ForcePop** child | TNA, Darkest Dungeon, Die Hard Trilogy |
| Scene + callout layers | A ForceBack and a ForcePopBack child with the **same rect**, so callouts never fight the scenes' priorities | Indiana Jones |
| Underlay | A ForceBack child under everything, or a `PuPFrames` image under a screen's own video | The Flintstones, Futurama |
| Text through a mask | A `PuPShapes` image on a ForcePop screen turns most of its video transparent | The Flintstones |
| Many tiny screens | One screen per digit, each fed a PNG by a trigger | Cuphead |

> **Rule of thumb.** Anything that must sit **above** a HUD label has to be a pop screen (ForcePopBack for "above video", ForcePop for "above everything"). Anything that must sit **below** it has to be video or an underlay. A frame on a **non-pop child** covers a **pop parent's** labels ([Darkest Dungeon](case-studies/darkest-dungeon.md)).

## Getting back to rest

Every pack needs a way to return a screen to its resting loop after an event. The idioms seen:

| Idiom | Effect | Seen in |
|---|---|---|
| SetBG once, then let clips end | The background restarts after every media ([chapter 6](06-playback-engine.md#the-background)) | most packs |
| `SetBackGround n,1` after a play | The clip just started becomes the new resting loop | Guardians of the Galaxy, Stranger Things |
| Stop, then SetBG the current mode loop | One routine is the only way back to rest | BTILC, Darkest Dungeon |
| A SetBG slideshow paced by a timer | Each step swaps the resting loop; real plays interrupt it | TNA |
| A stub clip at the loop's priority | An 85 ms clip replaces the loop, ends, and the background restarts | Stranger Things |
| A silent audio file | Replaces music at a higher priority (Normal) or as the new background (SetBG) | Transformers, Cuphead |
| StopFile on the exact file | Stops one exact clip and nothing else (its priority isn't checked) | King of the Hill, Transformers, BTILC |

Two traps:

- **SetBG waits behind a Loop.** A SetBG on a busy screen starts when the current media ends, and a Loop never ends. Stop the Loop first ([Transformers](case-studies/transformers.md)).
- **StopFile doesn't see backgrounds.** It only checks the current media, so it can't stop something started with SetBG ([TNA](case-studies/total-nuclear-annihilation.md)). And an **empty** StopFile consumes the playlist's next file instead of stopping anything ([Die Hard Trilogy](case-studies/die-hard-trilogy.md)).

## Clearing images

| Way | What happens | Seen in |
|---|---|---|
| A real transparent PNG | The label stays visible and shows nothing. The next image shows normally. | Darkest Dungeon (`Blank.png`), Cuphead, BTILC (`none.png`), Guardians of the Galaxy (`clear1.png`) |
| A name that doesn't exist | The label is cleared and **hidden** ([chapter 10](10-labels.md#image-captions)). For pBackground, later overlays don't show it again until `pDMDSetHUD 1` ([chapter 4](04-playlists.md#special-playlists)). | TNA, Guardians of the Galaxy, The Matrix |
| A tiny blank image at size 0 | Nothing is drawn | TNA |
| Show/hide with a `" "` caption | The image stays, only visibility changes | Futurama |

The missing-name idiom is fine for ordinary image labels. On **pBackground** it's how two packs lose their HUD frame for good: [The Matrix](case-studies/the-matrix.md) clears with `clear1.png` but ships `clear.png`, and [Batman 66](case-studies/batman-66.md) plays `joker.png` where only `joker1.png`… exist. Ship a transparent file under every name you use to clear a frame.

## Priorities

- **Ladders.** Most packs give each kind of content a band: attract low, mode progress in the middle, jackpots, drains, tilt and game over at the top ([King of the Hill](case-studies/king-of-the-hill.md), [Indiana Jones](case-studies/indiana-jones.md), [Darkest Dungeon](case-studies/darkest-dungeon.md)). Lower requests are **refused**, not queued ([chapter 6](06-playback-engine.md#priorities)), which is what a ladder wants.
- **Equal priority as a design.** [Futurama](case-studies/futurama.md) puts every trigger at 25 and lets the script decide; equal priority always replaces.
- **SkipSamePrty for bursts.** Progress clips that may arrive in quick succession use SkipSamePrty so they don't cut each other off ([BTILC](case-studies/btilc.md), [Indiana Jones](case-studies/indiana-jones.md)).
- **End clips.** A clip that ends a loop must have a priority **at least equal** to the loop. Lower is refused ([Stranger Things](case-studies/stranger-things.md)).
- **Menus.** Items cycled with Loop rows must share one priority. A lower-priority variant is dropped ([Die Hard Trilogy](case-studies/die-hard-trilogy.md)).

## Volume habits

- Volume 0 is 1 %, never silent ([chapter 7](07-volume-ducking-sounds.md#the-volume-model)). Packs that want the ROM's sound to dominate use 0 or 1 on nearly every row ([Indiana Jones](case-studies/indiana-jones.md), [Monster Bash](case-studies/monster-bash.md)).
- Ducking is done three ways: by file name (`-65_callout.mp3`, [Darkest Dungeon](case-studies/darkest-dungeon.md), [Die Hard Trilogy](case-studies/die-hard-trilogy.md)), by hand with FN 11 and a restore timer ([Batman 66](case-studies/batman-66.md), [Stranger Things](case-studies/stranger-things.md)), or entirely in VPX audio when the music isn't played by PUP ([Futurama](case-studies/futurama.md)).
- Music on its own MusicOnly screen, with callouts on another, keeps a callout from replacing the music.

## The usual bugs: a pre-release checklist

Every one of these was found in at least one shipped pack. All of them are logged ([chapter 16](16-troubleshooting.md)): reading the log after one game catches most of them.

**`triggers.pup`**

- [ ] Every PlayList names a folder **listed in `playlists.pup`**, spelled as on disk (`DrKahlsRobot` vs `DrKhalsRobot`, `AttackArbalest` vs `AttackArbelest`, `Bride Scream` vs `Bride_Bride Scream`).
- [ ] Every PlayFile exists in that folder (`Slurm18.mp4`, `raidersflashingarrow.mp4`, seven `Animated_Text` clips…). A row with a missing file is dropped at load.
- [ ] Rows with identical Trigger text on one screen are meant to fire together. A copy-pasted row that kept its old trigger plays two clips at once.
- [ ] Every StopFile names a file, and the file was started as main media, not SetBG.
- [ ] Every event the script sends has a row, and every row is sent: grep the script's event numbers against the file.
- [ ] Rows for Off screens are intended (they load but never play).

**Captures**

- [ ] Capture files are named `<number>.bmp`, with no other digits before `.bmp`.
- [ ] Captures were taken with the ROM language and settings the pack expects, and the pack says so.

**Labels and images**

- [ ] Every name used to clear a frame exists as a transparent image, or the HUD is shown again afterwards.
- [ ] Every font name matches a family or file name in `FONTS`. A screen number passed as a font name isn't a font.
- [ ] Labels on pages above 1 that should be visible after a page switch are created visible, or re-sent after the switch.

**Options and startup**

- [ ] The script's default layout matches the shipped default option ([Cuphead](case-studies/cuphead.md)).
- [ ] The root `.pup` files equal one of the option folders.
- [ ] `GetRoot` is read only after the pack is started ([Guardians of the Galaxy](case-studies/guardians-of-the-galaxy.md)).
- [ ] Every folder played from the script is listed in `playlists.pup` ([King of the Hill](case-studies/king-of-the-hill.md)).

## Things these packs ship that VPX doesn't use

- **Executables and batch files:** option `.bat` installers, `pupinit.bat`, `PuPDesktopPunch.exe`, Scorbit token and QR tools, iScored interfaces. Copy the option `.pup` files by hand ([chapter 2](02-pack-files.md#things-vpx-ignores)).
- **Instructions to delete the directb2s.** Not needed: PUP takes the backglass window when it has a usable screen for it ([chapter 1](01-concepts-and-setup.md#where-screens-appear-windows)).
- **`ExactColorMatch.txt`** in `PupCapture`: matching always uses dot shades ([chapter 5](05-triggers.md#dmd-frame-triggers)).
- **Window capture** of a PUP screen onto a playfield flasher (`VideoCapUpdate "PUPSCREEN18"`): there are no PUP desktop windows to capture ([The Matrix](case-studies/the-matrix.md)).
- **`.pup` files or pack copies outside the pack folder**, such as next to the `.vpx` or in a folder at the table root: only `pupvideos/<name>/` is read.
- **Window-flag CustomPos values** and patched-player readmes.
