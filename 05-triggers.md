# 5. Triggers

`triggers.pup` turns machine events into actions on screens.

```
ID,Active,Descript,Trigger,ScreenNum,PlayList,PlayFile,Volume,Priority,Length,Counter,RestSeconds,Loop,Defaults
1,1,Start BG,D0,2,Backglass,bg.mp4,,20,,,,SetBG,0
2,1,Ramp,W65,2,Ramps,,,30,,,5,,0
3,1,Multiball,"L42=1,S12=1",2,Multiball,mb.mp4,90,50,,,,Loop,0
4,1,----- MODES -----,,,,,,,,,,,0
```

Row 4 has an empty PlayList, so it's just a separator comment.

## Columns

At least **13** fields per line: the trailing `Defaults` column is optional. A shorter line is skipped, and logged by the screen it names (when that screen exists).

| # | Column (editor name) | Default | Meaning |
|---|---|---|---|
| 0 | `ID` | | Ignored. |
| 1 | `Active` | | The row loads only if this reads as the number `1` (`01` and ` 1` count). |
| 2 | `Descript` | | Free text (logs). |
| 3 | `Trigger` | | The condition list, see below. Quote it if it has commas. |
| 4 | `ScreenNum` (Screen) | | Screen the action applies to. |
| 5 | `PlayList` | | Playlist. **Empty = the row is a comment** and is ignored, except for `PlaySSF` and `CustomFunc` rows. For `CustomFunc`, this column holds the JSON. |
| 6 | `PlayFile` | | File in the playlist. Empty = the playlist's next file. For `PlaySSF`: the sound name. For `CustomFunc`: an optional message. |
| 7 | `Volume` | playlist volume | 1–200. **Empty means the playlist's volume** (from `playlists.pup`). `0` is not mute ([chapter 7](07-volume-ducking-sounds.md)). |
| 8 | `Priority` | 0 | See [chapter 6](06-playback-engine.md#priorities). `999` means stop. |
| 9 | `Length(s)` | 0 | Stop the media after this many seconds (0 = play to the end). |
| 10 | `Counter` | 0 | Skip-counter (see below). For `PlaySSF`: the `x,y,z` sound position. |
| 11 | `RestSeconds` | 0 | Positive: minimum seconds between two firings of this row. **Negative: a delay.** |
| 12 | `Loop` (PlayAction) | Normal | The action keyword. |
| 13 | `Defaults` | 0 | `1` = "simple play" (see below). |

### When rows are loaded

Each screen loads the rows for its own number **when the screen is created**. So:

- A row for a screen that doesn't exist is never loaded.
- The row's playlist must exist when the screen is created, and its PlayFile must exist in that playlist. Otherwise the row is dropped with a log line (`Playlist not found`, `PlayFile not found for playlist …`).
- With the `Init`-only scripted style ([chapter 1](01-concepts-and-setup.md#c-from-the-script-with-init-only)), `playlists.pup` isn't loaded, so `triggers.pup` rows referencing playlists can't load. That style is script-driven.

## Events

An event is a letter, a number and a value. These are the events VPX produces:

| Letter | Source | Number | Value |
|---|---|---|---|
| `W` | PinMAME switch | switch number | `1` closed, `0` open |
| `S` | PinMAME solenoid | solenoid number | the controller's state value |
| `L` | PinMAME lamp | lamp number | the controller's state value |
| `G` | PinMAME GI string | GI number | the controller's state value |
| `N` | PinMAME mech | mech number | the mech position (integer) |
| `D` | PinMAME segment display | element index, counted across all segment displays from 0 | bitmask of lit segments |
| `D` | DMD frame match | the matched frame's id | `1`, then immediately `0` |
| `D` | pack start | `0` | `1` (the `D0` startup event) |
| `E` | B2S controller generic data (`B2SSetData`/`B2SPulseData`) | data id | value |
| `B` | B2S controller score digit | digit | value |
| `C` | B2S controller score | | value |
| any | table script, `PuPlayer.B2SData "<letter><number>", value` | | as given |

- Machine states (`W S L G N`, segment `D`) are sent **only when they change**, polled about 60 times per second.
- When the event stream starts, every known state is sent once (states start as "unknown"). **Triggers on `=0` conditions fire at startup** for every switch or lamp that's off.
- `B2SData` events are sent **every time** the script calls it, changed or not.
- Mechs use the letter **`N`**. A trigger written `M5` never fires.

> **Rule.** For lamps, solenoids and GI, the value is whatever the controller reports. Plain on/off outputs report `1`/`0`. If a lamp trigger never fires, the output probably reports a different value; check it in a debug log ([chapter 16](16-troubleshooting.md)).

### Original tables: firing triggers from the script

Tables without a ROM drive the pack with `B2SData`. The PuPDMD framework wraps it:

```vbscript
Sub PuPEvent(EventNum)
    PuPlayer.B2SData "D" & EventNum, 1
End Sub

PuPEvent 105        ' fires every trigger row "D105" (or "D105=1")
```

`B2SData` takes the first character as the letter and the rest **must be a plain number**.

## Conditions

The `Trigger` column is a comma-separated list of conditions:

```
<letter><number>            means  <letter><number> = 1
<letter><number>=<value>
```

Examples: `W65`, `W65=0`, `L42=1`, `D105`, `"L42=1,S12=1"`, `"W30=1,D0=1"`.

- The **letter is case-sensitive**: `w65` never fires.
- The number is read as an integer. Leading zeros are fine (`W065` is `W65`). A non-numeric number never matches.
- A missing or non-numeric value means `1`.
- **No spaces.** `"W5, W6"` has a condition whose letter is a space, and the row never fires.
- There's no limit on the number of conditions.

### Matching rules

1. Rows are grouped per screen by their **exact Trigger text**. All rows on the same screen with identical text share one set of condition values, and they **all fire together**, in file order. This is how one event plays a video *and* sets a frame on the same screen.
2. Rows on different screens, or with different text (`W5` vs `W5=1`), are independent, even if they mean the same thing.
3. Each group remembers the last value it saw for each of its conditions. All values start at 0.
4. When an event arrives, every group containing that letter+number stores the new value. If **all** of the group's conditions now equal their expected values, the group fires.
5. **There's no edge detection.** A group fires on every event that leaves it satisfied, even if the value didn't change. Sending `B2SData "D5",1` twice fires twice.
6. **`D` values are events, not states.** After a group fires, every `D` condition in it is reset to 0. So `"D9=1,W5=1"` fires when D9 arrives while W5 is 1 (or when W5 becomes 1 after D9 arrived), and doesn't fire again on a later W5=1 unless D9 is sent again.
7. Conditions on other letters keep their values. `"L42=1,S12=1"` fires when the second of the two becomes true, and fires again on any later event for L42 or S12 while both are still 1.

> **Rule.** Different screens can fire in any order for the same event. Don't rely on screen 2's row running before screen 5's.

## What happens when a row fires

Each firing goes through these steps, per row:

1. **Off screen.** If the screen is `Off`, nothing happens.
2. **Delay** (RestSeconds negative). The row is scheduled to run later instead of now:
   - `-1` to `-98` → that many **seconds**,
   - `-99` or lower → that many **milliseconds** (`-500` → 0.5 s, `-1500` → 1.5 s).

   While a delayed run is pending, further firings of the row are **ignored** (not counted, not queued). When the delay ends, the row continues from step 3 and **skips the rest check** of step 4.
3. **Counter.** With `Counter` = N ≥ 2, the row acts on firings 1, N+1, 2N+1… and skips the others. **It does not act "every Nth time starting at N"**: `Counter 3` acts on hits 1, 4, 7, 10. Counter 0 or 1 acts every time.
4. **Rest** (RestSeconds positive). If the row acted less than RestSeconds ago, it's skipped. The rest timer is set when the row acts, whether or not the screen actually started anything.
5. **Action**, per the `Loop` column.

The playlist has its own rest time too ([chapter 6](06-playback-engine.md#the-rest-gate)). A row can pass its own rest check and still be refused by the playlist's.

## Actions (`Loop` column)

Keywords are case-insensitive. Anything not in this table, including an empty column, is `Normal`.

| Keyword | Action | Details in |
|---|---|---|
| *(empty)*, `Normal`, anything else | Play the file | [Chapter 6](06-playback-engine.md#normal) |
| `Loop` | Play the file looping | [Chapter 6](06-playback-engine.md#loop) |
| `SplashReset` | Play over the current media, which restarts afterwards | [Chapter 6](06-playback-engine.md#splashreset-and-splashresume) |
| `SplashResume` (also `SplashReturn`) | Play over the current media, which continues where it was afterwards | [Chapter 6](06-playback-engine.md#splashreset-and-splashresume) |
| `StopPlayer` | Stop the screen if Priority allows | [Chapter 6](06-playback-engine.md#stopping) |
| `StopFile` | Stop the screen if it's playing exactly this file | [Chapter 6](06-playback-engine.md#stopping) |
| `SetBG` | Make the file the screen's background loop | [Chapter 6](06-playback-engine.md#the-background) |
| `SkipSamePrty` | Play, unless something of the same (non-zero) priority is playing | [Chapter 6](06-playback-engine.md#priorities) |
| `QueueSameP` | Queue behind the current media (if priority allows) | [Chapter 6](06-playback-engine.md#queue-actions) |
| `QueueAlways` | Queue behind the current media, whatever its priority | [Chapter 6](06-playback-engine.md#queue-actions) |
| `PlaySSF` | Play a sound effect | [below](#playssf) |
| `CustomFunc` | Run a custom function on the screen | [below](#customfunc) |

> **Rule (spelling).** The keyword is **`SkipSamePrty`**: `SkipSamePriority` is not recognized and behaves as `Normal`. `SplashReturn` is accepted as a synonym of `SplashResume`.

### Defaults = 1: simple play

When `Defaults` is `1` (and the action isn't PlaySSF or CustomFunc), the row's own settings are replaced. It plays the **next file** of its playlist at the **screen's default volume**, priority 0, action Normal, no length. PlayFile, Volume, Priority, Length and the action keyword are all ignored, so a `StopPlayer` row with `Defaults 1` *plays* instead of stopping. The row's Counter and RestSeconds still apply.

### PlaySSF

```
10,1,Pop sound,W40,2,,boom.wav,80,,,"-5,1,0",,PlaySSF,0
```

- **PlayFile** is the sound's file name in `PUPSOUNDS`, **including `.wav`**.
- **Counter** must be `x,y,z` (quoted). With fewer than three values the row is dropped at load.
- **Volume** empty means 100.
- PlayList, Priority and Length are ignored. RestSeconds (rest or delay) applies.
- Only `x` matters: it pans from −10 (left) to 10 (right). See [chapter 7](07-volume-ducking-sounds.md#sound-effects-pupsounds).
- A ball-roll sound name (`ballroll0.wav`…) **stops** that ball-roll slot, since a trigger has no speed value.

### CustomFunc

```
20,1,Hide HUD,D300,5,"{'FN':3,'OT':0}",,,,,,,CustomFunc,0
21,1,Hide + duck,D301,5,"{'FN':3,'OT':0}","{'mt':301,'SN':2,'FN':42,'DV':30}",,,,,,CustomFunc,0
```

- **PlayList** holds the custom function JSON ([chapter 9](09-sendmsg-custom-functions.md)). It is applied to **this row's screen**. Any `SN` inside is ignored.
- **PlayFile**, if not empty, is then sent as a full `SendMSG` message, so it must be a JSON object with an `mt` (for example an `mt 301` aimed at another screen, or an `mt 402` sound).
- Counter and RestSeconds apply. Volume, Priority and Length are ignored.
- Use **single quotes** inside the JSON. The CSV reader strips double quotes, though the JSON reader also accepts the unquoted result.

CustomFunc rows work whether or not the table script creates a `PinUpPlayer.PinDisplay` object, so they also work in packs for pure PinMAME tables.

## DMD frame triggers

For PinMAME games with a DMD, a pack can fire `D<n>` triggers when a specific DMD frame appears.

- Put the reference frames in the pack's **`PupCapture`** folder as `<n>.bmp`. A frame on screen matching `123.bmp` fires `D123` (value 1, then 0).
- Matching uses the largest DMD of the game that is at least 128 pixels wide, in 2- or 4-bit-plane formats:
  - **128×32**: matched as is.
  - **128×16**: placed in the middle rows (8–23) of a 128×32 frame.
  - **192×64 / 256×64**: scaled down by 2 with a 2×2 average. 256×64 gives 128×32, and 192×64 gives a 96×32 image centered in 128×32.
- Each new DMD frame is matched once.

How a capture matches:

- Captures must be **uncompressed 24-bit 128×32** BMPs, the size of the frame described above. 128×16 and 192×64 files are read but never match; other files are skipped.
- The ID is the **number right before `.bmp`**, case-insensitive. `123.bmp`, `Cap_123.bmp` and `Jackpot 123.BMP` are all ID 123. Two files with the same ID (for example `111.bmp` and `233346391 Cap_111.bmp`) keep only one of them, and which one is unspecified.
- The red channel is read as the shade of each dot. A rectangle **outlined in magenta (253, 0, 253)** limits the match to the dots inside it: everything outside the rectangle (a score, for example) is ignored. Without a rectangle, the whole frame must match.
- Captures are tried in **ascending ID order**, and the first that matches decides. Lower IDs win when two captures match the same frame.
- The **same capture never fires twice in a row**. It fires again only after a different capture has matched in between. There is no time-based re-arm.
- `ExactColorMatch.txt` is ignored. Matching always uses the dot shades.

> **Rule.** Name captures with the trigger number only (`111.bmp`). Any digits just before `.bmp` become the ID, so a descriptive name ending in digits can silently create or replace a capture.

- **Serum colorizations** can identify frames instead. When a Serum colorization is active for the game, it reports the triggers and VPX's own matching is switched off (so triggers never fire twice).

If the pack has `D` triggers (other than `D0`) but neither a `PupCapture` set nor an active Serum colorization, the log says:

```
This PuP pack has N DMD trigger(s) but no trigger data of its own, and no Serum colorization is identifying frames for '<rom>'. Those triggers cannot fire.
```

That's the usual cause of a pack that loads and shows its backgrounds, but never plays its event videos. It was built against a colorization that isn't installed.
