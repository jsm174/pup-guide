# 16. Troubleshooting

## Read the log first

Almost every PUP failure is logged. Open the VPX log after a run and look for lines from the PUP plugin. Useful lines at normal log level:

| Log line | Meaning |
|---|---|
| `PUP path: …` | The pack folder that was found. If it's missing, no pack was found ([chapter 1](01-concepts-and-setup.md#how-vpx-finds-a-pack)). |
| `No global PUP folder configured; per-table 'pupvideos' used when present` | Normal when `PUPFolder` is empty. |
| `PUP folder was not found (settings is '…')` | `PUPFolder` is set but has no `pupvideos` inside. |
| `Screen added: screen={…}` | One per screen, with mode, volume multiplier and CustomPos. Check modes here. |
| `Invalid screen mode: …` | Typo or stray space in the mode column. The screen is Off. |
| `Failed to parse screen line, expected at least 6 columns…` | Too few columns in a `screens.pup` line. |
| `Invalid playlist: …` | Too few columns in a `playlists.pup` line. |
| `Failed to parse trigger line, expected at least 13 columns…` | Too few columns in a `triggers.pup` line. |
| `Playlist folder not found: …` | Folder in `playlists.pup` doesn't exist. |
| `Duplicate playlist: …` | Same folder listed twice. The second is skipped (harmless). |
| `Playlist folder … is empty` | |
| `Playlist not found: …` | A trigger row or a script call names a playlist that isn't loaded. |
| `PlayFile not found for playlist …` | A trigger row names a file that doesn't exist. The row is dropped. |
| `Font added: familyName=…, styleName=…, filename=…` | One per font: the names you can use in `LabelNew`. |
| `Font not found: label=…, font=…` | The label will draw no text ([chapter 10](10-labels.md#fonts)). |
| `Duplicate label: …` | `LabelNew` with an existing name. Ignored. |
| `Invalid label: …` | `LabelSet` on a label that doesn't exist (logged once per label). |
| `Image not found, hiding label: …` | Image caption doesn't exist (often on purpose, the "clear.png" idiom). |
| `Unable to open: filename=…` | The media file can't be opened: corrupt, or not media. |
| `This PuP pack has N DMD trigger(s) but no trigger data of its own…` | DMD triggers can't fire ([chapter 5](05-triggers.md#dmd-frame-triggers)). |
| `PinUpPlayer was instantiated and initialized from the table script before table init…` | The script started PUP itself, so the automatic load was skipped. |
| `Not implemented: …`, `… not implemented` | The pack or script uses a feature VPX doesn't support. Usually harmless. |

Raising the log level to **debug** adds the play decisions: `MAIN PLAY`, `BG PLAY`, `Dropping lower priority`, `Resting playlist, skipping`, `Skipping resting trigger`, `Skipping trigger on Off screen`, `MAIN SKIP (same file)`, and `Processing trigger`.

## Symptoms

| Symptom | Likely cause |
|---|---|
| Nothing at all on any window | No pack found (no `PUP path:` line); plugin disabled; windows not enabled in VPX; FFmpeg libraries missing. |
| Backglass shows B2S instead of PUP | Screen 2 (and 6) unusable: missing, Off, MusicOnly, hidden (`Hide`/FN 28), or placed inside another screen. |
| ScoreView shows screen 1 instead of 5 | Screen 5 is unusable (missing, Off, MusicOnly, hidden) or has a CustomPos pointing elsewhere. |
| A screen never appears | Its number has no window and no CustomPos (chapter 3); its mode is Off (check `Screen added`); stray space in the mode. |
| Background never starts | No `D0` trigger and no `screens.pup` background; SetBG on an `Off` screen. |
| Background starts but is silent | Started by `screens.pup` (volume 0 → 1 %) or by `playlistplayex …,0,…`. Use a `D0` SetBG trigger with a volume. |
| **No trigger ever fires** | `triggers.pup` lines with fewer than 13 columns; `Active` isn't 1; screen number not defined; playlist name has a space; lowercase letter; `M` instead of `N`. |
| Lamp/solenoid trigger never fires | The output reports a value other than 1 ([chapter 5](05-triggers.md#events)). |
| DMD-frame videos never play | No `PupCapture` set and no Serum colorization. |
| One DMD-frame trigger fires on the wrong frame, or never | Two capture files with the same trailing number (`111.bmp`, `Cap_111.bmp`); a lower-numbered capture also matches that frame; the capture isn't 128×32 ([chapter 5](05-triggers.md#dmd-frame-triggers)). |
| A DMD-frame trigger fires only once | The same capture doesn't fire twice in a row; another capture must match in between. |
| A mode loop keeps playing after a SetBG | SetBG waits for the current media to end, and a Loop never ends. Stop the loop first ([chapter 6](06-playback-engine.md#the-background)). |
| A StopFile row does nothing | The file was started with SetBG (StopFile only checks the current media), or the row has no file name ([chapter 6](06-playback-engine.md#stopping)). |
| A trigger fires twice | Two rows with identical trigger text on the same screen; `B2SData` sent twice; a Serum colorization and something else both sending. |
| A video doesn't restart when retriggered | Same file already playing: by design ([chapter 6](06-playback-engine.md#evaluation-order)). |
| Videos only play sometimes | Playlist rest time (default 30 s if the column is empty); trigger RestSeconds; Counter; lower priority than what's playing. |
| Videos cut each other off | Equal priority replaces. Use `SkipSamePrty` or higher priorities. |
| Queued video never plays | The current media is looping. |
| Splash doesn't resume where it was | The row uses `SplashReset` (restarts) instead of `SplashResume`. |
| A picture (png) doesn't show | The screen has a background video, and static images only show when no video plays. Use PuPOverlays or an image label. |
| The overlay shows a **file name as text** | A non-image file (`.webp`…) played through PuPOverlays/PuPAlphas. |
| Text labels are invisible | Font not in `FONTS`; empty font name; label on another page; caption empty; FN 3 off; FN 34 pending; overlay faded out by `pDMDScreenFadeOut`. |
| Labels vanished and never come back | FN 34 (`pHideOverlayDuringCurrentPlay`) sent while nothing plays on the screen: it waits for the next end of media. |
| Splash page stays up for minutes | `PDMDSplashPagePlaying(HUD)` with no video ending: 500 s fallback. |
| HUD frame (pBackground) gone for good after some event | A PuPOverlays or caption name that doesn't exist was set on pBackground (often a "clear" name, or a bare name where only numbered files exist). Ship the file, or send `pDMDSetHUD 1` ([chapter 4](04-playlists.md#special-playlists)). |
| HUD frame (pBackground) gone after a splash | A `hidehudplay` splash returned by timer or by another page change instead of a video end. Send `pDMDSetHUD 1`. |
| Fade-in label ends hidden | It was started while a fade-out was still running ([chapter 12](12-label-animations.md#rules-common-to-every-animation)). |
| Labels stuck half-transparent | An overlay fade (`pDMDScreenFadeOut/In`) was interrupted by a page switch. Send FN 22 `AM:0`. |
| Clone labels appear in the top-left corner | `pDMDLabelClone` sends percent where pixels are expected ([chapter 13](13-pupdmd-framework.md#animations)). |
| Label text too big after `pDMDLabelSetSizeText` | `size` is a point size, bigger than `fonth`. |
| Label at the wrong place | Position 0 means 50 % for centered labels; positions are anchors; `v2` missing or present. |
| Shadow/outline not showing | No shadow type set; offsets 0. |
| A label won't change from image to text | Image labels stay images ([chapter 10](10-labels.md#image-captions)). |
| A label won't hide | It's animating; it's on another page and the caption isn't `` `u` ``. |
| Labels reappear after hiding them | Page switch to a page > 1 resets them to their created visibility. |
| Sound effect silent | Name without `.wav`; file not directly in `PUPSOUNDS`; not a `.wav`; volume 0; script uses SoundAdd/SoundPlay. |
| Everything else stays quiet after a callout | A duck that never ended (`ALL:1`, or an FN 42 from a screen with nothing playing). Send FN 42 `DV:-1`. |
| Hidden screen still audible | By design: `Hide`/FN 28 only stop drawing. Use FN 12 (Off) or `playstop` for silence ([chapter 3](03-screens-and-layout.md#changing-modes-at-runtime)). |
| Works on Windows, not on Mac/Linux | Fonts only installed on Windows; option `.bat` never ran; case differences and `\` separators are fine. |
