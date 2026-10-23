# 7. Volume, ducking and sounds

## The volume model

Each play request carries a **requested volume** `v` (see the table in [chapter 6](06-playback-engine.md#where-play-requests-come-from)). It becomes an output level in three steps:

1. `v = −1` means "the playlist's volume" (`playlists.pup` column 5).
2. `v` is limited to **1–200**. **0 becomes 1.**
3. The output level is

   ```
   output % = MainVol × clamp( round(v × M / 100), 1, 100 )
   ```

   where `M` is the screen's **volume multiplier** (`screens.pup` column 6, default 100, or FN 26).

Consequences:

- **Volume 0 is not mute.** The quietest a request can be is 1 %. `playlistplayex n,"x","y.mp4",0,1` (a very common idiom for "silent background") plays at 1 %.
- **Nothing goes above 100 %.** Values above 100 only matter when the multiplier is below 100: `v` 200 with `M` 50 gives 100 %.
- The only real silence is `MainVol` = 0, or a media file without an audio track. Background videos should be encoded **without audio**.
- Each media keeps its own requested volume `v`. `MainVol` (the plugin setting) and the multiplier are applied when the media **starts**, and `MainVol` also applies live to what's playing. A queued item or a background therefore plays at the `MainVol` and multiplier of the moment it starts, not of the moment it was requested.

### Default and current volume

| Value | Set by | Used for |
|---|---|---|
| **Default volume** (per screen, initially 100) | `setVolume n,v` (`pDMDVolumeDef`), FN 10 (all screens, `VL`) | `play`, `playlistplay`, trigger rows with `Defaults 1`. It doesn't change what's playing now. |
| **Current volume** (per screen) | the volume of the media that starts, every time one starts (a new media, a queue advance, a background coming back, each pass of a loop); `setVolumeCurrent n,v` (`pDMDVolumeCurrent`), FN 11, FN 40 (`VL`) | The media playing right now. `setVolumeCurrent` also becomes that media's own volume, so its later loop passes keep it. The next media brings its own volume. |
| **Multiplier** | `screens.pup` column 6, FN 26 (`VL`) | Everything on the screen: what's playing now, and every media from its start on. |

Re-requesting the file that's already playing doesn't restart it, but it does apply the new request's volume, and that becomes the media's own volume for its later passes.

## Ducking

Ducking temporarily lowers the other screens while one screen plays something important (a callout over the music).

### Starting a duck

**By file name.** When a media whose file name starts with `-` starts playing (as current media or background):

| File name | Duck level |
|---|---|
| `-30callout.mp4` | 30 % |
| `-00callout.mp4` | 1 % |
| `-callout.mp4` (no two digits) | the screen's default duck level, set with FN 54 `DV`. **With no FN 54 there is no default, and nothing is ducked.** |

**By custom function.** FN 42:

```vbscript
PuPlayer.SendMSG "{ ""mt"":301, ""SN"":5, ""FN"":42, ""DV"":30 }"            ' duck all others to 30 %
PuPlayer.SendMSG "{ ""mt"":301, ""SN"":5, ""FN"":42, ""DV"":30, ""ALL"":1 }"  ' duck every screen, 5 included
PuPlayer.SendMSG "{ ""mt"":301, ""SN"":5, ""FN"":42, ""DV"":-1 }"             ' restore now
```

`DV` 0 or missing means 1 %.

### What a duck does

- Every **other** screen that is **currently playing** (current media or background) fades from the volume of what it's playing down to `DV` % of that volume, by 5 volume units every 100 ms (or every `FI` ms after an [FN 29](06-playback-engine.md#fading-a-screen-out-fn-29) on that screen).
- Screens that are idle at that moment aren't ducked. Media they start later plays at full volume.
- The source screen becomes the **duck source**. While any screen is the duck source, **new duck requests are ignored** (except `DV -1`).

### Ending a duck

- When the **duck source's** media **ends** ([chapter 6](06-playback-engine.md#when-media-ends): its end, a stop, a queue advance or a loop pass), every ducked screen fades back to the volume it was playing at by 6 units per step, on the same 100 ms timer (or jumps back if it isn't playing anymore).
- `DV -1` restores immediately.
- A ducked screen that **starts** a media (a new media, a queue advance, its background coming back, or a loop pass) **forgets** its duck: that media plays at its own volume, and the later restore leaves it alone.
- `setVolumeCurrent` on a ducked screen cancels the fade on that screen.

> **Rule.** A `-NN` file that loops (a looping media or a background) releases the duck at the end of each pass and ducks again when the next pass starts, so the other screens briefly come back up between passes. An FN 42 duck from a screen that has nothing playing waits for the next end of media on that screen. `DV -1` always ends a duck.

> **Rule.** With `ALL:1` the source is ducked too and is **not** made the duck source, so nothing ever restores the duck automatically. Always pair `ALL:1` with a later `DV:-1`.

## Fade-in of new media

On the 2.0 engine, FN 55 (or FN 56) `FS` sets a per-screen **visual** fade-in for new videos: opacity starts at `FS`/255 and rises by `FS`/255 per drawn frame. `FS` 255 disables it (the default), and so does 0. A negative `FS` is ignored. `FS` 10 is a soft fade of about 25 frames. It affects the picture, not the audio, and only current-media starts, not background restarts. On the 1.5 engine FN 55 and FN 56 are ignored.

For a fade-**out** of a whole screen, picture and sound, see FN 29 in [chapter 6](06-playback-engine.md#fading-a-screen-out-fn-29).

## Sound effects (PUPSOUNDS)

Separate from video audio, a pack can have a bank of short sound effects.

### The bank

- Every `.wav` file **directly** in the pack's `PUPSOUNDS` folder is loaded (decoded to memory) when the pack folder is set. Other formats and subfolders are ignored.
- A sound is named by its **file name including `.wav`**, case-insensitively: `boom.wav`, `BOOM.WAV`.
- Files named `ballroll` + a number + `.wav` also fill the **ball-roll slots** (below): `ballroll0.wav` … `ballroll28.wav`, so slots 0–28.

### Playing

```vbscript
PuPlayer.PuPSound "boom.wav", -5, 0, 0, 80, ""                 ' name, x, y, z, volume, special
PuPlayer.SendMSG "{'mt':402,'ns':'boom.wav','sX':-5,'sY':0,'sZ':0,'vol':80,'sp':''}"
```

and from triggers with the `PlaySSF` action ([chapter 5](05-triggers.md#playssf)), and `playevent` type 7 (centered, volume −1 → 100).

- **Position:** only `x` is used. It pans with constant power from **−10** (full left) through **0** (center) to **10** (full right). `y` and `z` are accepted and ignored. On the 2.0 engine an `x` beyond ±19 is divided by 10 first, so a −100…100 range works too.
- **Volume:** percent, 0–200 (above 100 amplifies). **0 plays nothing.** `MainVol` doesn't apply to sound effects. On the 2.0 engine the volume is first multiplied by the **sound effect volume** (FN 62 `SF`, percent, default 100, for the whole pack whatever the `SN`) and then limited to 0–100.
- **1.5 engine:** each call starts a new one-shot playback. Overlapping plays of the same sound are fine.
- **2.0 engine:** a sound that's played again **restarts**: the previous playback of that name is cut off.
- An unknown name does nothing (and isn't logged).
- Sounds are always panned in stereo. The 2.0 engine's multi-channel (surround) panning isn't emulated.

### Ball roll

**1.5 engine:** if the name starts with `ballroll` + a digit and that slot (0–9, the first digit) is loaded:

- `special` is a **speed**, a whole number: `"0"` (or empty) stops the slot, any other number starts it looping if it isn't already running.
- Every call updates the slot's volume and pan, which take effect at the next loop pass.

**2.0 engine:** any name starting with `ballroll` is a ball-roll call, never a plain sound. The number after it (with or without `.wav`) is the slot, 0–28. A slot that isn't loaded does nothing.

- `special` is a **speed** (decimals allowed): `0` (or empty) stops the slot, any other value starts it looping if it isn't already running.
- The speed also sets the **pitch**: the sample plays at (|speed| / 100 × 0.75 + 0.25) × 44100 Hz, so speed 100 is normal pitch for a 44.1 kHz file and small speeds play it at a quarter of that.
- Every call updates the slot's volume, pan and pitch, which take effect at the next loop pass.

```vbscript
PuPlayer.PuPSound "ballroll0.wav", 0, 0, 0, 60, "5"   ' start rolling
PuPlayer.PuPSound "ballroll0.wav", 0, 0, 0, 60, "0"   ' stop
```

### Ignored sound calls

`SoundAdd`, `SoundPlay`, and `mt` 400/401 messages are accepted and **ignored**. Sounds can only come from `PUPSOUNDS`.

### Audio output

Sound effects and media audio both go to the "PinUp Player" audio source, routed to VPX's backglass audio device.
