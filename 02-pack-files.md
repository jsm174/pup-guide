# 2. Pack folder and file format

## Folder layout

```
pupvideos/
  <game>/                    ← the pack folder (ROM name or the name given to B2SInit)
    screens.pup              ← screens and their modes            (chapter 3)
    playlists.pup            ← which subfolders are playlists     (chapter 4)
    triggers.pup             ← event → action rules               (chapter 5)
    displays.pup             ← optional: live DMD overlays        (chapter 14)
    FONTS/                   ← .ttf / .otf fonts for labels       (chapter 10)
    PUPSOUNDS/               ← .wav sound effects                 (chapter 7)
    PupCapture/              ← <n>.bmp DMD frames that fire D<n>  (chapter 5)
    PUPDisplays/             ← glass images used by displays.pup  (chapter 14)
    PuPOverlays/             ← special playlist: label-layer overlay images
    PuPFrames/               ← special playlist: underlay images
    PuPAlphas/               ← special playlist: same as PuPOverlays
    PuPShapes/               ← special playlist: transparency masks
    <any playlist folder>/   ← media: mp4, mov, mkv, webm, mp3, wav, png, jpg, gif...
```

None of these files is strictly required. A pack with no `screens.pup` loads with no screens, which is fine for script-only packs that create screens with `Init`.

### Things VPX ignores

Windows-era packs ship extras that VPX never reads or runs:

- `pupinit.bat`, `*.bat`, `*.exe`, `usePuPDriver.ini` are not executed.
- Option folders such as `PuP-Pack_Options/` with alternate `.pup` sets. On Windows a `.bat` copies the chosen set over the main files. Elsewhere you copy the `.pup` files of the option you want into the pack folder yourself.
- `ScreenType.txt` and similar files are ignored by VPX, but a table **script** may read them (for example through `PuPlayer.GetRoot`).

## Media formats

Videos and audio are decoded by FFmpeg, so practically any container and codec FFmpeg knows will play (mp4/H.264, mkv, mov, webm, avi, mp3, ogg, wav…). There is no extension whitelist.

- **Alpha video works.** Frames are converted to RGBA, so a codec that decodes with an alpha channel (for example ProRes 4444 in `.mov`) is drawn with transparency over what's below it.
- Video is always **stretched** to the screen rectangle. There is no aspect-ratio preservation (`setAspect` is not supported).
- In a normal playlist, `png`, `jpg`, `jpeg` and `bmp` files are shown as **static images** without going through the video decoder ([chapter 6](06-playback-engine.md#images-in-a-normal-playlist)). `gif`, `webp` and the rest go through FFmpeg like video.
- A file with only an audio stream plays as audio. Any frame held on screen from the previous video is dropped.

## The `.pup` file format

All three `.pup` files use the same simple comma-separated format, and it's stricter than it looks.

> **Rule 1: the first line is always skipped.** It's treated as the header whatever it contains. A pack whose first line is a real screen loses that screen.

> **Rule 2: minimum column counts.** A line with fewer fields than the minimum is skipped and logged (a short trigger line only when the screen it names exists). Missing trailing columns take their defaults, and extra columns are ignored:
>
> | File | Fields | Columns |
> |---|---|---|
> | `screens.pup` | at least **6** | 8 (Priority and CustomPos optional) |
> | `playlists.pup` | at least **7** | 7 |
> | `triggers.pup` | at least **13** | 14 (Defaults optional) |

> **Rule 3: no spaces around commas.** Only the whole line is trimmed. Fields are not. `2, ForceBack` gives the mode `" ForceBack"`, which is not a valid mode, so the screen becomes `Off`. A playlist name `" Jackpot"` doesn't exist, and a trigger token `" W5"` has the letter `' '`. Numeric fields are trimmed when parsed, so stray spaces there are harmless.

> **Rule 4: quotes group, and vanish.** A double quote switches quoted mode on and off and is removed from the value. Commas inside quotes are kept. There is no escape for a literal quote: `""` is just two toggles and produces nothing. So
> `"5,0,0,100,100"` → `5,0,0,100,100`, and
> `"{""FN"":3}"` → `{FN:3}` (which still works as JSON for custom functions, see [chapter 5](05-triggers.md#customfunc)).

> **Rule 5: empty lines are skipped, and there is no comment syntax.** A trigger row with an empty PlayList is the closest thing to a comment ([chapter 5](05-triggers.md)).

Other details:

- **Encoding:** each line is read as UTF-8, or as Windows-1252 if it isn't valid UTF-8. A UTF-8 BOM is ignored.
- **Line endings:** CRLF or LF.
- **Numbers:** a numeric field that is empty or not a number takes the column's default (listed per file in the following chapters).
- **Names:** playlist folders, file names, label names and font names are matched case-insensitively. Folders and files on disk are also found case-insensitively, which matters on Linux and macOS.

## Paths and separators

- In **label captions**, both `\` and `/` work, repeated separators collapse, and leading separators are ignored, so the `"PuPOverlays\\frame.png"` that VBScript scripts often contain works, and so does `"\\PuPOverlays\\frame.png"`.
- In **play file names** (`triggers.pup` PlayFile, `playlistplayex`, `playevent`), a file in a subfolder of a playlist is given relative to the playlist folder, with either separator: playlist `Jackpot`, file `alt\j1.mp4` or `alt/j1.mp4`.
- Playlist folders are direct subfolders of the pack folder.
