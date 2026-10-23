# 4. Playlists

A playlist is a **subfolder** of the pack folder. Playing "from a playlist" means playing either a named file in it or its **next** file.

## `playlists.pup`

```
ScreenNum,Folder,Des,AlphaSort,RestSeconds,Volume,Priority
0,Background,,0,0,100,0
0,Jackpots,Jackpot callouts,1,10,80,0
```

At least 7 fields per line.

| # | Column (editor name) | Default | Meaning |
|---|---|---|---|
| 0 | `ScreenNum` | | **Ignored.** Every playlist is available on every screen. |
| 1 | `Folder` (Playlist) | | Folder name inside the pack, found case-insensitively. This is also the playlist's name. |
| 2 | `Des` (Description) | | Free text. |
| 3 | `AlphaSort` (Randomize) | 0 | `1` shuffles. **Any other value, including 2, means alphabetical.** |
| 4 | `RestSeconds` | 30 | Minimum seconds between two plays from this playlist (see [the rest gate](06-playback-engine.md#the-rest-gate)). Empty or not a number means **30**, not 0. `0` or negative means no rest. |
| 5 | `Volume` | 100 | The playlist volume, used when a play asks for "playlist volume" (volume −1, or an empty trigger Volume column). |
| 6 | `Priority` | 0 | Read and stored, **not used** by playback. |

> **Rule.** The rest default is 30 seconds. A playlist row with an empty RestSeconds column holds back repeated normal plays for 30 s. Write `0` explicitly for no rest.

What happens at load:

- **Missing folder:** logged (`Playlist folder not found`) and skipped. Triggers using it are dropped.
- **Empty folder:** logged as a warning and kept. Requests for its next file fail.
- **Duplicate folder** (same name, any case): the second row is logged (`Duplicate playlist`) and skipped.

## Each screen has its own copy

Every screen gets its **own copy** of every playlist. So each screen keeps its own "next file" position and its own rest timer for the same folder. Playing `Jackpots` on screen 2 doesn't advance or rest `Jackpots` on screen 5.

Playlists added from the script with `playlistadd` exist only on that one screen, with volume 100.

## Which files are in the rotation

The folder is scanned **once**, at load (or at `playlistadd`). Files added later are not seen until the pack reloads.

- Files directly in the folder join the rotation if they have a media extension: `mp4`, `mp3`, `avi`, `mkv`, `mov`, `asf`, `vob`, `ogg`, `flv`, `wav`, `png`, `gif`, `jpg`, `jpeg`, `m4v`, `webm`, `wmv`, `mpg`, `mpeg`, `m4a`, `aac`, `flac`, `bmp`, `webp`, `apng`. Hidden files (names starting with `.`, like `.DS_Store`) and other files (`Thumbs.db`, `readme.txt`…) are left out.
- Files left out of the rotation, and files in **subfolders**, can still be played by name, with a relative path for subfolders (`alt/j1.mp4`, see [chapter 2](02-pack-files.md#paths-and-separators)).
- **Order:** sorted by file name using a plain byte comparison, so uppercase sorts before lowercase (`B.mp4` before `a.mp4`) and `10.mp4` before `2.mp4`. Zero-pad numbered files (`001.mp4`, `002.mp4`…) to control the order.
- **Shuffle** (`AlphaSort` = 1) happens **once**, at load. After that the shuffled order repeats in a fixed cycle until the pack reloads. There's no reshuffle on wrap.
- The **next file** position starts at the first file and wraps silently. It advances when a play with an empty file name **passes the priority and rest checks** ([chapter 6](06-playback-engine.md#evaluation-order)). A request rejected by priority or rest doesn't consume a file. One that passes and is then skipped (because the same file is already playing) does. If the next file is an image (`png`, `jpg`, `jpeg`, `bmp`), it's taken right away, without priority or rest checks, even on an `Off` screen ([chapter 6](06-playback-engine.md#images-in-a-normal-playlist)).
- A `StopFile` with an empty file name also takes the next file to compare against, so it **advances the rotation** too.

## Looking up files

- A named file is found by its path **relative to the playlist folder**, case-insensitively: `jackpot1.MP4` finds `Jackpot1.mp4`.
- A trigger whose PlayFile doesn't exist in its playlist is **dropped at load**, with `PlayFile not found for playlist …` in the log.
- A script play of a missing file logs `PlayFile not found` and does nothing.

## Special playlists

Four folder names, matched case-insensitively, turn a playlist into something other than media. They only work when they're **listed in `playlists.pup`** (or added with `playlistadd`) like any other playlist. Playing a file from them (trigger, `playlistplayex`, `play`, `playevent`) does this:

| Folder name | Effect of playing a file from it |
|---|---|
| `PuPFrames` | The image becomes the screen's **underlay**, stretched to the screen, drawn **under** the video. It stays until another frame replaces it. Since it's under the video, it only shows where no video is drawn: when idle, or through masked or transparent areas. |
| `PuPOverlays` | The image becomes the image of the screen's **pBackground label** ([chapter 10](10-labels.md#pbackground)): stretched over the whole screen, **above** the video, below all other labels. It follows pBackground's visibility, so `pDMDSetHUD 0` and `hidehudplay` hide it. |
| `PuPAlphas` | Exactly the same as `PuPOverlays`. |
| `PuPShapes` | The image becomes the screen's video **mask**: every pixel with the same color as the image's top-left pixel `(0,0)` becomes transparent in the video, and the rest stays. The mask is scaled to the video frame and stays until another shape replaces it. It applies to video only, not to images or labels. |

These plays **bypass the whole playback engine**: no priority, rest, queue or volume, and they don't interrupt the current video.

> **Rule (name the file).** Special playlists have no rotation: always give the file name. With an empty name, PuPOverlays/PuPAlphas set the caption to the bare folder name, which is drawn as text.

> **Rule (missing overlay).** A missing PuPOverlays/PuPAlphas file hides pBackground if it was showing an image, and a later valid overlay image does **not** make it visible again. Show it with `pDMDSetHUD 1`.

> **Rule (overlay image types).** Because PuPOverlays/PuPAlphas files go through the label system, they must have a label image extension: `png`, `jpg`, `jpeg`, `bmp`, `gif`, `apng`, `agif`. Anything else, such as a `.webp`, is taken as **text**. If pBackground was showing text (it starts empty), the file *name* is then drawn as text across the screen. If it already held an image, the image stays and the call is ignored.

> **Rule.** PuPFrames and PuPShapes images are loaded by file. Any common image type works (`png`, `jpg`, `bmp`, `tga`, `webp`…).

## Images in normal playlists

`png`, `jpg`, `jpeg` and `bmp` files played from an ordinary playlist (by name or as the next file) are not videos. They become the screen's **static image**, covered in [chapter 6](06-playback-engine.md#images-in-a-normal-playlist).
