# Monster Bash

A PinMAME table with the simplest layout in this collection: **one screen per window**, a frame PNG, and 84 capture triggers. It shows how far rest-gated Normal rows go without any splash or queue machinery.

| | |
|---|---|
| ROM and pack folder | `mb_106b` (WPC95, 128×32) |
| Started by | PinMAME, automatically |
| Installed option | Option 1 "BG, DMD, Topper" |
| Backglass | screen 2 |
| Topper | screen 0 |
| ScoreView | not PUP |

## Screen layout

| # | Mode | Role |
|---|---|---|
| 0 | ForceOn | Topper; background `Topper/Topper_Frame1.mp4` |
| 2 | ForceOn | Everything; background `BGDefault/BG_Default1.mp4` |

On screen 2, `D0` sets the pBackground to `Metal.png`, a thin metal frame drawn over the video.

The table also has a directb2s. Its B2S controller can report an **empty** game name before the ROM is known; VPX ignores a controller with an empty game name, so the pack is found from the ROM name.

## How media is driven

- 87 active rows, all on screen 2: `D0` for the frame, `D1`–`D84` (one row each), `W11` (launch button) and `W28` (left eject).
- 84 captures, IDs 1–84 exactly: every D number has a capture and every capture has a row. Half use a magenta rectangle to limit the match to part of the frame.
- **Priorities 1–10:** attract and game over 1, mode progress 2–4, completions 5–7, jackpots 6–9, drains and specials 10.
- **Trigger rest on almost every row** (mostly 1–5 s, 60 s on game start), on top of the playlists' own rest.
- **Volume as a mixer:** about two-thirds of rows use volume 0 (1 %) and rely on the ROM's speech; music clips are pushed to 125–150. Output never exceeds 100 %, so those just play at full volume ([chapter 7](../07-volume-ducking-sounds.md#the-volume-model)).

### Same-file attract music

Two attract frames, D51 and D67, both request `Attract_Music.mp4`. Requesting the file that's already playing doesn't restart it ([chapter 6](../06-playback-engine.md#evaluation-order)), so repeated attract frames keep one continuous music track instead of stuttering.

### Calm by rest

The creature clips D1–D3 and D5 share one playlist at priority 3. A second clip while the first plays is refused while the playlist rests; once the screen is idle, the next one plays. Clips on **different** playlists at the same priority replace each other. Without a single Splash or Queue action, the backglass stays calm.

## What doesn't work

- **Two playlist names don't match their folders:** `Bride Scream` (folder `Bride_Bride Scream`) and `Mummy_Mummy Mayhem` (folder `Mummy Mayhem`). Both rows log "Playlist not found" and never fire.
- **`D0` has Counter 6.** `D0` fires once, and the first hit always acts ([chapter 5](../05-triggers.md#what-happens-when-a-row-fires)), so this does nothing.
- Five empty playlists, and the two option folders listed as playlists.

## Lessons

- Copy playlist names from the folder list. One typo silently kills a capture-driven event, and only the log tells you.
- For a single-screen backglass, a frame PNG plus rest-gated Normal rows gives a calm show with nothing else.
