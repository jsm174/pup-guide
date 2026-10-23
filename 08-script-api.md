# 8. Script API reference

The table script talks to PUP through the `PinUpPlayer.PinDisplay` object:

```vbscript
Dim PuPlayer
Set PuPlayer = CreateObject("PinUpPlayer.PinDisplay")
```

VPX provides this object itself. No Windows COM component is involved. All methods that take a screen number log `Screen not found` and do nothing when the screen doesn't exist.

Tables may create several of these objects; they all talk to the same loaded pack. Releasing an object doesn't unload the pack.

## Overview

| Method / property | Status |
|---|---|
| [`Init`](#init) | supported |
| [`B2SInit`](#b2sinit) | supported |
| [`playlistadd`](#playlistadd) | supported |
| [`playlistplay`](#playlistplay) | supported |
| [`playlistplayex`](#playlistplayex) | supported |
| [`play`](#play) | supported |
| [`playevent`](#playevent) | supported (types 0–8, 11, 12) |
| [`playpause`, `playresume`, `playstop`](#playpause-playresume-playstop) | supported |
| [`SetLength`, `SetLoop`, `SetBackGround`](#setlength-setloop-setbackground) | supported |
| [`setVolume`, `setVolumeCurrent`](#setvolume-setvolumecurrent) | supported |
| [`isPlaying`](#isplaying) | supported (read) |
| [`Show`, `Hide`](#show-hide) | supported |
| [`SetScreenEx`](#setscreenex) | mode only |
| [`B2SData`](#b2sdata) | supported |
| [`SendMSG`](#sendmsg) | `mt` 301, 402 (400/401 ignored) |
| [`LabelInit`, `LabelNew`, `LabelSet`, `LabelShowPage`](#labels) | supported ([chapter 10](10-labels.md)) |
| [`PuPSound`](#pupsound-soundadd-soundplay) | supported |
| `SoundAdd`, `SoundPlay` | accepted, ignored |
| [`GetRoot`](#getroot) | supported (read) |
| [`GetVersion`](#getversion) | returns `"2.0.99"` |
| [`CloseApp`, `PuPClose`](#closeapp-pupclose) | supported |
| `setWidth`, `setHeight`, `setPosX`, `setPosY`, `setAspect` | not supported (logged) |
| `SetScreen`, `BlockPlay`, `SN`, `B2SFilter`, `GetGame`, `B2SDisplays`, `InitPuPMenu`, `SetPosVideo`, `LabelSetEx`, writing `isPlaying` / `GetRoot` | not supported (logged) |
| `GameUpdate` | not supported, returns 0 |
| `GrabDC`, `GrabDC2` | not available |

## Starting and stopping

### Init

```vbscript
PuPlayer.Init screen, "folder"
```

Sets the pack folder (loading `FONTS` and `PUPSOUNDS`) and creates `screen` as a plain `Show` screen. If the screen already exists, it is **replaced**. Doesn't read `screens.pup`/`playlists.pup` and doesn't start the trigger event stream. See [the Init trap](01-concepts-and-setup.md#c-from-the-script-with-init-only).

Classic scripted setup:

```vbscript
PuPlayer.Init pTopper, "mySample"
PuPlayer.Init pBackglass, "mySample"
PuPlayer.playlistadd pTopper, "TopperBack", 1, 0
PuPlayer.playlistadd pBackglass, "Ramps", 1, 30
PuPlayer.playlistplay pTopper, "TopperBack"
PuPlayer.SetBackGround pTopper, 1
```

### B2SInit

```vbscript
PuPlayer.B2SInit "", "folder"
```

Full pack load ([chapter 1](01-concepts-and-setup.md#what-a-full-load-does-in-order)). The first argument is ignored. An empty folder name does nothing. Calling it again for the same folder keeps everything and makes sure triggers are running. A different folder unloads the old pack.

### CloseApp, PuPClose

Unload the pack: all screens, labels, playlists, fonts and sounds. The pack also unloads by itself when the game ends.

## Playlists and playing

### playlistadd

```vbscript
PuPlayer.playlistadd screen, "folder", sort, restSeconds
```

Adds a playlist to **this screen only**, with volume 100. `sort`: 0 alphabetical, **any non-zero value shuffles** (unlike `playlists.pup`, where only 1 shuffles). Adding a playlist the screen already has is logged and ignored, so you can't override a `playlists.pup` playlist's rest time this way.

### playlistplay

```vbscript
PuPlayer.playlistplay screen, "playlist"
```

Plays the playlist's **next** file at the screen's default volume, priority 0, Normal. The playlist rest time applies.

### playlistplayex

```vbscript
PuPlayer.playlistplayex screen, "playlist", "file.mp4", volume, priority
```

The main "play this" call. `file` empty = next file. `volume` −1 = the playlist's volume, 0 = 1 % ([chapter 7](07-volume-ducking-sounds.md)). `priority` as in [chapter 6](06-playback-engine.md#priorities), 999 = stop. Action Normal.

```vbscript
PuPlayer.playlistplayex pDMD, "Jackpots", "", 100, 50        ' next jackpot, priority 50
PuPlayer.playlistplayex pDMD, "PuPFrames", "frame2.png", 0, 1 ' set an underlay (volume/priority ignored)
PuPlayer.playlistplayex pDMD, "Misc", "", 0, 999             ' stop what's playing (not a background-only screen)
```

### play

```vbscript
PuPlayer.play screen, "playlist", "file.mp4"
```

Plays the file (empty = next) at the screen's default volume, priority 0, Normal.

### playevent

```vbscript
PuPlayer.playevent screen, "playlist", "file.mp4", volume, priority, playtype, seconds, ""
```

The full-featured play. `playtype` is the action number:

| playtype | Action |
|---|---|
| 0 | Normal |
| 1 | Loop |
| 2 | SplashReset |
| 3 | SplashResume |
| 4 | StopPlayer: stop if `priority` is 0 or above the current one. Playlist/file ignored. |
| 5 | StopFile: stop if the current media is this file |
| 6 | SetBG |
| 7 | PlaySSF: plays sound `file` from PUPSOUNDS, centered, at `volume` (negative → 100) |
| 8 | SkipSamePrty |
| 9 | not supported (logged) |
| 10 | not supported (logged) |
| 11 | QueueSameP |
| 12 | QueueAlways |

`seconds`: the request's length. Above 0, the media ends that many seconds after the request ([chapter 6](06-playback-engine.md#length-loop-and-pause-from-the-script)); for an image, it's the display time. It only applies to this request: a refused or queued request doesn't touch what's playing. The last argument (special) is ignored.

### playpause, playresume, playstop

```vbscript
PuPlayer.playpause screen
PuPlayer.playresume screen
PuPlayer.playstop screen
```

Pause and resume the current video. `playstop` stops the screen: current media and queue dropped, background restarted ([chapter 6](06-playback-engine.md#stopping)).

### SetLength, SetLoop, SetBackGround

```vbscript
PuPlayer.SetLength screen, seconds    ' end what's playing N seconds from now (0 = now)
PuPlayer.SetLoop screen, 1            ' 1 loop what's playing, 0 stop looping
PuPlayer.SetBackGround screen, 1      ' 1 current media becomes the background, 0 clear it
```

`SetLength` and `SetLoop` act on what's playing **right now**, background included ([chapter 6](06-playback-engine.md#length-loop-and-pause-from-the-script)). `SetBackGround …,1` copies the current media into the background slot and `…,0` clears it ([the background](06-playback-engine.md#the-background)); on ForcePop, ForcePopBack and MusicOnly screens the slot always ends up empty.

### setVolume, setVolumeCurrent

```vbscript
PuPlayer.setVolume screen, 80          ' default volume for later play/playlistplay (1..200)
PuPlayer.setVolumeCurrent screen, 30   ' volume of what's playing now (1..200)
```

### isPlaying

```vbscript
If PuPlayer.isPlaying(pDMD) Then ...
```

True while the screen has a **current media** that's actually playing. A screen showing only its background returns **false**.

## Screens

### Show, Hide

```vbscript
PuPlayer.Show screen
PuPlayer.Hide screen
```

`Hide` stops drawing the screen; its media keeps playing and is still heard. `Show` draws it again. Neither changes the screen's mode ([chapter 3](03-screens-and-layout.md#changing-modes-at-runtime)).

### SetScreenEx

```vbscript
PuPlayer.SetScreenEx screen, x, y, width, height, popup
```

Only `popup` is used: 0 → Show, 1 → ForcePop, 2 or more → MusicOnly. Position and size are ignored (logged when both width and height are non-zero).

## Events

### B2SData

```vbscript
PuPlayer.B2SData "D105", 1
PuPlayer.B2SData "E12", 0
```

Sends a trigger event. The first character is the letter (case-sensitive) and the rest must be a number. It's delivered even when the trigger event stream isn't running, and every call is an event, even with the same value ([chapter 5](05-triggers.md#matching-rules)).

## SendMSG

```vbscript
PuPlayer.SendMSG "{ ""mt"":301, ""SN"":5, ""FN"":3, ""OT"":0 }"
PuPlayer.SendMSG "{'mt':301,'SN':5,'FN':3,'OT':0}"
```

JSON messages, single or double quotes. Covered fully in [chapter 9](09-sendmsg-custom-functions.md).

## Labels

`LabelInit`, `LabelNew`, `LabelSet`, `LabelShowPage`: see [chapter 10](10-labels.md).

## PuPSound, SoundAdd, SoundPlay

```vbscript
PuPlayer.PuPSound "name.wav", x, y, z, volume, "special"
```

Plays a `PUPSOUNDS` sound ([chapter 7](07-volume-ducking-sounds.md#sound-effects-pupsounds)). `SoundAdd` and `SoundPlay` are ignored.

## Information

### GetRoot

```vbscript
path = PuPlayer.GetRoot
```

- With a pack loaded: the folder **containing** the pack, with a trailing separator (`…/pupvideos/`), so `PuPlayer.GetRoot & "BTILC\ScreenType.txt"` builds a path into the pack.
- With no pack loaded: the global `pupvideos` folder from the settings (empty if none).

### GetVersion

Returns `"2.0.99"`. Scripts that check for "PinUP 1.5 or later" or "2.0 features" get the newest code path.
