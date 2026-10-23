# The PinUP Pack Guide for Visual Pinball X

This guide documents how PinUP packs ("PuP packs") behave in Visual Pinball X, which plays them through its built-in **PUP plugin**. It is meant for:

- **pack authors** building or porting a PuP pack,
- **table scripters** driving PinUP from VBScript (`PinUpPlayer.PinDisplay`, the PuPDMD framework),
- **players and cabinet builders** who want to know why a pack looks or sounds the way it does.

PinUP has a lot of unwritten rules. A value of `0` that doesn't mean zero, a "seconds" argument that turns into milliseconds above 500, a trigger that fires twice, a label that won't change, a video that won't restart. This guide tries to write every one of them down. Each chapter covers one subsystem in full. [Chapter 15](15-rules-cheatsheet.md) collects all the surprising rules in one checklist.

## Chapters

| # | Chapter | What's in it |
|---|---|---|
| 1 | [Concepts and setup](01-concepts-and-setup.md) | Mental model, how VPX finds and starts a pack, settings, windows |
| 2 | [Pack folder and file format](02-pack-files.md) | Folder layout, special folders, the `.pup` CSV dialect |
| 3 | [Screens and layout](03-screens-and-layout.md) | `screens.pup`, screen numbers, modes, CustomPos, which window shows what, draw order |
| 4 | [Playlists](04-playlists.md) | `playlists.pup`, folder scanning, order, rest time, special playlists |
| 5 | [Triggers](05-triggers.md) | `triggers.pup`, events (W/S/L/G/N/D/E…), conditions, firing rules, counter, rest, delay |
| 6 | [The playback engine](06-playback-engine.md) | Priorities, actions, the queue, splash, background, stop, images, frames, masks, end of media |
| 7 | [Volume, ducking and sounds](07-volume-ducking-sounds.md) | Volume math, ducking, fade, the PUPSOUNDS bank, ball roll |
| 8 | [Script API reference](08-script-api.md) | Every `PinUpPlayer.PinDisplay` method and property |
| 9 | [SendMSG and custom functions](09-sendmsg-custom-functions.md) | `mt` messages, every `FN` custom function |
| 10 | [Labels: the text and image overlay](10-labels.md) | Overlay, pBackground, fonts, LabelNew, LabelSet, captions, images, pages |
| 11 | [Label special properties (`mt:2`)](11-label-properties.md) | Every property key |
| 12 | [Label animations (`mt:1`)](12-label-animations.md) | Flash, move, zoom, pulse, fade, wiggle, filter flash, clone, number count-up |
| 13 | [The PuPDMD framework](13-pupdmd-framework.md) | What each `pDMD…` helper sub actually sends, and what it really does |
| 14 | [Authored DMD overlays (`displays.pup`)](14-dmd-overlays.md) | Putting a live DMD on top of PUP screens |
| 15 | [The rules cheatsheet](15-rules-cheatsheet.md) | Every quirk in one place |
| 16 | [Troubleshooting](16-troubleshooting.md) | Symptom → cause table, reading the log |
| 17 | [Case studies: what real packs do](17-case-studies.md) | Fifteen real packs: how they're driven, layering recipes, idioms, a pre-release checklist, and one page per pack (including a full walk-through of BTILC) |

## Conventions used in this guide

- **Screen** means a PinUP display number (0, 1, 2, 5, 11…), not a physical monitor. Physical outputs are called **windows** (Backglass, ScoreView, Topper).
- Colors are **Windows BGR integers**: `0x00BBGGRR`, which is what VBScript's `RGB(r,g,b)` returns. Pure red is `255`, pure blue is `16711680`.
- `%` positions and sizes are percentages of the **screen's own rectangle** unless stated otherwise.
- File, playlist, label and font names are **case-insensitive** everywhere unless a rule says otherwise. Trigger letters and a few keywords are case-sensitive, and those cases are called out.
- **Rule** boxes mark behavior that is easy to get wrong.

> **Rule.** When this guide says something "does nothing", it usually still writes a line to the VPX log. Read the log first when something doesn't work ([chapter 16](16-troubleshooting.md)).

## A 60-second example

A minimal pack for a PinMAME table with ROM `mytable_l1`:

```
<table folder>/
  mytable.vpx
  pupvideos/
    mytable_l1/
      screens.pup
      playlists.pup
      triggers.pup
      Background/  loop.mp4
      Jackpot/     jackpot1.mp4  jackpot2.mp4
```

`screens.pup`
```
ScreenNum,ScreenDes,PlayList,PlayFile,Loopit,Active,Priority,CustomPos
2,Backglass,Background,loop.mp4,0,ForceBack,,
```

`playlists.pup`
```
ScreenNum,Folder,Des,AlphaSort,RestSeconds,Volume,Priority
2,Background,,0,0,100,0
2,Jackpot,,1,5,100,0
```

`triggers.pup`
```
ID,Active,Descript,Trigger,ScreenNum,PlayList,PlayFile,Volume,Priority,Length,Counter,RestSeconds,Loop,Defaults
1,1,Jackpot lamp,L45,2,Jackpot,,,0,,,,,0
```

When PinMAME starts `mytable_l1`, VPX finds the folder and loads the pack. `loop.mp4` loops as the backglass background at near-silent volume. Each time lamp 45 turns on, the next jackpot video plays (shuffled, at most once every 5 seconds). When it ends, the background comes back.

> **Rule.** The 5-second rest only applies because the trigger priority is 0. Any priority above what is currently playing skips the playlist rest time completely ([chapter 6](06-playback-engine.md#the-rest-gate)).
