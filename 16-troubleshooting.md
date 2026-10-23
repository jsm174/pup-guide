# 16. Troubleshooting

## Read the log first

Almost every PUP failure is logged. Open the VPX log after a run and look for lines from the PUP plugin. Useful lines at normal log level:

| Log line | Meaning |
|---|---|
| `PUP path: …` | The pack folder that was found. If it's missing, no pack was found ([chapter 1](01-concepts-and-setup.md#how-vpx-finds-a-pack)). |
| `PinUP engine: 1.5` / `PinUP engine: 2.0 (PinUpDisplayX2)` | Which engine the pack's `usePuPDriver.ini` selected ([chapter 1](01-concepts-and-setup.md#the-engine-pinup-15-or-20)). |
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
| `Unscaled label canvas is drawn scaled` | FN 53 asked for an unscaled label canvas of another size than the screen. Labels are drawn scaled to the screen anyway. |

Raising the log level to **debug** adds the play decisions: `MAIN PLAY`, `BG PLAY`, `Dropping lower priority`, `Resting playlist, skipping`, `Skipping resting trigger`, `Skipping trigger on Off screen`, `MAIN SKIP (same file)`, and `Processing trigger`.

## Symptoms

| Symptom | Likely cause |
|---|---|
| Nothing at all on any window | No pack found (no `PUP path:` line); plugin disabled; windows not enabled in VPX; FFmpeg libraries missing. |
| Backglass shows B2S instead of PUP | Screen 2 (and 6) unusable: missing, Off, MusicOnly, hidden (`Hide`/FN 28), or placed inside another screen. |
| ScoreView shows screen 1 instead of 5 | Screen 5 is unusable (missing, Off, MusicOnly, hidden), or is placed inside screen 0, 1, 2 or 6: then it's drawn only in that screen's window ([chapter 1](01-concepts-and-setup.md#where-screens-appear-windows)). |
| A screen never appears | Its number has no window and no CustomPos (chapter 3); its mode is Off (check `Screen added`); stray space in the mode. |
| Background never starts | No `D0` trigger and no `screens.pup` background; SetBG on an `Off` screen; SetBG with a lower priority than what was playing (refused). |
| Background starts but is silent | Started by `screens.pup` (volume 0 → 1 %) or by `playlistplayex …,0,…`. Use a `D0` SetBG trigger with a volume. |
| **No trigger ever fires** | `triggers.pup` lines with fewer than 13 columns; `Active` isn't 1; screen number not defined; playlist name has a space; lowercase letter; `M` instead of `N`. |
| Lamp/solenoid trigger never fires | The output reports a value other than 1 ([chapter 5](05-triggers.md#events)). |
| DMD-frame videos never play | No `PupCapture` set and no Serum colorization. |
| One DMD-frame trigger fires on the wrong frame, or never | Two capture files with the same trailing number (`111.bmp`, `Cap_111.bmp`); a lower-numbered capture also matches that frame; the capture isn't 128×32 ([chapter 5](05-triggers.md#dmd-frame-triggers)). |
| A DMD-frame trigger fires only once | The same capture doesn't fire twice in a row; another capture must match in between. |
| A mode loop keeps playing after a SetBG | The SetBG's priority is lower than the loop's, so it's refused. Give the SetBG at least the loop's priority, or stop the loop first ([chapter 6](06-playback-engine.md#the-background)). |
| A screen's labels vanish while one of its clips plays, and `LabelSet` does nothing | The clip's path contains `@`, which hides the screen's labels until the clip ends ([chapter 10](10-labels.md#hiding-labels-during-a-media)). |
| Labels (scores, frame) vanish while another screen plays | That screen's video is in the always-on-top band (ForceON, ForcePoP, FN 4 `FS:1`) and came to the front after the labels. Use a normal-band mode for it, or have the labels' screen start a media or FN 6 afterwards ([chapter 3](03-screens-and-layout.md#draw-order)). |
| A StopFile row does nothing | The file was started with SetBG (StopFile only checks the current media), or the row has no file name ([chapter 6](06-playback-engine.md#stopping)). |
| A trigger fires twice | Two rows with identical trigger text on the same screen; `B2SData` sent twice; a Serum colorization and something else both sending. |
| A video doesn't restart when retriggered | Same file already playing: by design ([chapter 6](06-playback-engine.md#evaluation-order)). |
| Videos only play sometimes | Playlist rest time (default 30 s if the column is empty); trigger RestSeconds; Counter; lower priority than what's playing. |
| Videos cut each other off | Equal priority replaces. Use `SkipSamePrty` or higher priorities. |
| Queued video never plays | The current media is looping. |
| Splash doesn't resume where it was | The row uses `SplashReset` (restarts) instead of `SplashResume`. |
| A picture (png) doesn't show | The screen's background came back (after a stop or a later media end), and static images only show when no video plays. Use PuPOverlays or an image label. Outside custom image mode: refused by priority or rest like a video. |
| A picture disappears after 10 s, or after a stop | 1.5 engine without custom image mode: the image plays as a 10 s media. Send FN 7 `CP:1` (or FN 36) to the screen first ([chapter 6](06-playback-engine.md#images-in-a-normal-playlist)). |
| A picture only flashes | 2.0 engine outside custom image mode (after FN 7 `CP:0` or FN 12), or a `bmp` on the 2.0 engine: a one-frame media. Use png/jpg and keep custom image mode on. |
| A behaviour differs from Windows PinUP | Check the `PinUP engine:` log line: the pack may need the other engine's `usePuPDriver.ini` ([chapter 1](01-concepts-and-setup.md#the-engine-pinup-15-or-20)). |
| The overlay shows a **file name as text** | A non-image file (`.webp`…) played through PuPOverlays/PuPAlphas. |
| Text labels are invisible | Font not in `FONTS`; empty font name; label on another page; caption empty; FN 3 off; FN 34 pending; overlay faded out by `pDMDScreenFadeOut`; 2.0 engine: overlay created while the screen was hidden or an idle pop screen (it shows at the next media start); FN 57 `OH:1` with another video drawn above the screen's. |
| Label text smaller or larger than expected | 2.0 engine: text is scaled by the screen's shape (a 4:3 screen gives 0.75) or by FN 58 / FN 53 ([chapter 10](10-labels.md#text-height-on-the-20-engine)). |
| Labels or a hidden screen come back on their own | Any media start (a loop pass of the background too) shows a screen hidden by FN 28 and turns FN 3 `OT:0` back on. Use FN 2 `FO:0` to keep the overlay off. |
| Label animations too fast or too slow | FN 44/47 paint interval other than 32 ms on the screen ([chapter 12](12-label-animations.md#rules-common-to-every-animation)). |
| Labels vanished and never come back | FN 34 (`pHideOverlayDuringCurrentPlay`) sent while nothing plays on the screen: it waits for the next end of media. |
| Splash page stays up for minutes | `PDMDSplashPagePlaying(HUD)` with no video ending: 500 s fallback. |
| HUD frame (pBackground) gone for good after some event | A PuPOverlays or caption name that doesn't exist was set on pBackground (often a "clear" name, or a bare name where only numbered files exist). Ship the file, or send `pDMDSetHUD 1` ([chapter 4](04-playlists.md#special-playlists)). |
| HUD frame (pBackground) gone after a splash | A `hidehudplay` splash returned by timer or by another page change instead of a video end. Send `pDMDSetHUD 1`. |
| Fade-in label ends hidden | It was started while a fade-out was still running ([chapter 12](12-label-animations.md#rules-common-to-every-animation)). |
| Labels stuck half-transparent | An overlay fade (`pDMDScreenFadeOut/In`) was interrupted by a page switch. Send FN 22 `AM:0`. |
| Clone labels appear in the top-left corner | `pDMDLabelClone` sends percent where pixels are expected ([chapter 13](13-pupdmd-framework.md#animations)). |
| Label text too big after `pDMDLabelSetSizeText` | `size` is a point size, bigger than `fonth`. |
| Label at the wrong place | Position 0 means 50 % for centered labels; positions are anchors; `v2` missing or present. |
| Shadow/outline not showing | No shadow type set; offsets 0; shadow type 5 or more on the 2.0 engine. |
| Every new label has a shadow (or none) | An FN 38 earlier in the session set the default for labels created afterwards. |
| A label won't change from image to text | Image labels stay images ([chapter 10](10-labels.md#image-captions)). |
| A label won't hide | It's animating; it's on another page and the caption isn't `` `u` ``. |
| Labels reappear after hiding them | Page switch to a page > 1 resets them to their created visibility. |
| Sound effect silent | Name without `.wav`; file not directly in `PUPSOUNDS`; not a `.wav`; volume 0; script uses SoundAdd/SoundPlay; 2.0 engine: FN 62 `SF:0`, or a name starting with `ballroll` that isn't a loaded slot. |
| A repeated sound cuts itself off | 2.0 engine: playing a sound again restarts it ([chapter 7](07-volume-ducking-sounds.md#playing)). |
| Everything else stays quiet after a callout | A duck that never ended (`ALL:1`, or an FN 42 from a screen with nothing playing). Send FN 42 `DV:-1`. |
| Video fade-in (FN 55) does nothing | 1.5 engine: FN 55 and FN 56 only work on the 2.0 engine. |
| A screen stops playing anything | FN 24 safe mode (1.5 engine) is on: send FN 24 `EN:0`. Or FN 300: nothing restarts until the next play. |
| A full-screen effect is under the HUD instead of over it | 2.0 engine with CustomPos flag 6: the screen's media is drawn into its parent's video, below the parent's labels ([chapter 3](03-screens-and-layout.md#hosted-screens-flag-6-20-engine)). |
| Hidden screen still audible | By design: `Hide`/FN 28 only stop drawing. Use FN 12 (Off) or `playstop` for silence ([chapter 3](03-screens-and-layout.md#changing-modes-at-runtime)). |
| Works on Windows, not on Mac/Linux | Fonts only installed on Windows; option `.bat` never ran; case differences and `\` separators are fine. |
