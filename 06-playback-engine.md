# 6. The playback engine

Every screen runs its own media engine. This chapter gives its exact rules: what happens to a play request, how priorities, the queue, splashes and the background interact, and what happens when media ends.

## A screen's state

| Part | What it is |
|---|---|
| **Current media** | The media playing now (the head of the queue), with its priority, volume, length and loop flag. |
| **Queue** | Media waiting behind the current one. |
| **Background** | A file looped whenever the queue is empty. |
| **Underlay** | A `PuPFrames` image under the video. |
| **Static image** | A `png/jpg/jpeg/bmp` played from a normal playlist in custom image mode, shown when no video is playing. |
| **Mask** | A `PuPShapes` transparency mask applied to video frames. |

A screen is **playing** when it has a current media. **The background does not count**: a screen showing only its background loop is "idle" for priorities, `isPlaying` and queue actions. The **current priority** is the current media's priority, or 0 when idle.

## Where play requests come from

| Source | File | Volume | Priority | Action | Length |
|---|---|---|---|---|---|
| Trigger row | PlayFile (empty = next) | Volume column (empty = playlist volume) | Priority | `Loop` column | Length |
| Trigger row with `Defaults 1` | next | screen default | 0 | Normal | 0 |
| `playlistplay n, list` | next | screen default | 0 | Normal | 0 |
| `play n, list, file` | file (empty = next) | screen default | 0 | Normal | 0 |
| `playlistplayex n, list, file, vol, pri` | file (empty = next) | vol (−1 = playlist volume) | pri | Normal | 0 |
| `playevent n, list, file, vol, pri, type, secs, ""` | file (empty = next) | vol (−1 = playlist volume) | pri | type | secs |
| `screens.pup` PlayList column | PlayFile (empty = next) | 0 (→ 1 %) | −1 | SetBG | 0 |

The "screen default volume" is 100 unless changed with `setVolume` or FN 10.

## Evaluation order

A request goes through these steps, in this order. "Refused" means nothing changes on the screen. On the 1.5 engine, a screen in FN 24 safe mode refuses every request before step 1 ([safe mode](#safe-mode-and-hard-stop)).

**At the screen:**

1. **Special playlists.** If the playlist is `PuPFrames`, `PuPOverlays`, `PuPAlphas` or `PuPShapes`, the file is applied as an underlay, overlay or mask ([chapter 4](04-playlists.md#special-playlists)). Done. No priority, queue or volume involved.
2. **Images in custom image mode.** If the screen is in custom image mode and the file (the requested one, or the playlist's next file) ends in `png`, `jpg`, `jpeg` (or `bmp` on the 1.5 engine), see [images](#images-in-a-normal-playlist). Done. No priority involved. Outside custom image mode, images go through the steps below like any media.
3. **Off screen.** If the screen is `Off`, refused.
4. If the media engine accepts the request below, the screen's static image is cleared.
5. **Volume** is resolved: −1 becomes the playlist volume, then the value is limited to 1–200.

**In the media engine:**

6. **Priority 999** (any action, SetBG included): if a current media plays, stop the screen ([stopping](#stopping)). Either way, play nothing. Done.
7. **Lower priority**: if the screen is playing and the request's priority is **lower** than the current priority, refused, unless the action is `QueueAlways`. This applies to SetBG too: a refused SetBG doesn't change the background.
8. **SkipSamePrty**: if the screen is playing, the priority is above 0 and equal to the current priority, refused.
9. **Rest gate**: see [below](#the-rest-gate).
10. **File lookup**: an empty file name takes the playlist's next file (advancing the rotation). A missing file is logged and refused.
11. **SetBG** is handled ([the background](#the-background)). Done.
12. **Same file**: if the current media is the same file, it is **not restarted**. Only its volume is updated. Done.
13. **Background already showing it**: if the screen is idle and its background loop is playing this very file, refused (the playlist rest timer is still restarted).
14. **Queue actions** while playing: inserted behind the current media ([queue actions](#queue-actions)). Done.
15. **Splash actions**: played in front of the current media, which is kept ([splash](#splashreset-and-splashresume)). Done.
16. **Everything else**: the current media is discarded and the new one plays now.

Each accepted request (from step 10 on) also restarts the playlist's rest timer.

Every media that starts on a drawn screen (a play, a background start or restart, each pass of a loop, a static image) also shows the screen again if it was hidden, turns its label overlay back on (unless FN 2 `FO:0`) and changes the screen's place in the [draw order](03-screens-and-layout.md#draw-order).

## Priorities

- Priorities are plain integers. The usual range is 0–99. `999` means "stop".
- A request with priority **equal to or higher than** the current one **replaces** it. Equal priority interrupts, because "same priority" is not a reason to refuse (use `SkipSamePrty` for that).
- A **lower** priority is refused while something is playing. It doesn't wait. Use `QueueAlways` to make it wait.
- When the screen is idle (nothing, or only the background), the current priority is 0, so every request passes.
- The background never has a priority. Anything interrupts it.

> **Rule.** Priority 0 against priority 0 replaces. Two priority-0 triggers firing in a row cut each other off, and only the playlist rest time slows that down.

## The rest gate

A playlist's `RestSeconds` ([chapter 4](04-playlists.md)) only holds back requests that are:

- action **Normal** (not Loop, Splash, Queue, SkipSamePrty, SetBG…), **and**
- **not preempting**. A request preempts when its priority is above 0 **and** above the current priority.

So:

| Request | Screen | Rest checked? |
|---|---|---|
| Normal, priority 0 | idle | **yes** |
| Normal, priority 0 | playing priority 0 | **yes** |
| Normal, priority 10 | idle (current 0) | no: 10 > 0 preempts |
| Normal, priority 10 | playing priority 10 | **yes**: equal doesn't preempt |
| Normal, priority 10 | playing priority 20 | (refused by priority first) |
| Loop / SkipSamePrty / Queue / Splash, any priority | any | no |

> **Rule.** On an idle screen, **any positive priority skips the playlist rest time**. If you want a playlist to rest, play it at priority 0, or use the trigger row's own RestSeconds, which always applies.

The trigger-level `RestSeconds` ([chapter 5](05-triggers.md#what-happens-when-a-row-fires)) is a separate timer per row, checked before any of this.

## Actions

### Normal

Replaces the current media and plays now. Media waiting in the queue stays queued behind the new one.

### Loop

Like Normal, but the media restarts at its end forever. While a media loops, **nothing queued behind it ever plays**. Break the loop with `SetLoop n,0` (the media then ends normally and the queue continues), a stop, or a replacing request.

A `Length` on a looping media ends it after that time, whatever the pass.

On the 2.0 engine, a Loop request for a media **shorter than 0.9 s** (or whose length the file doesn't give) plays it **once**, like Normal. This applies when the media starts, not to backgrounds or to a later `SetLoop n,1`.

**Safe loop (FN 41).** FN 41 on a screen that plays something (current media or background) makes that media, if it loops and is longer than 1 s, wrap back to its start **1 s before its end** instead of playing to the end. A wrap made this way is **not** an end of media (no loop pass for FN 34, `@`, ducks, splash pages, jukebox). It lasts until another file is played on the screen.

### SplashReset and SplashResume

The splash plays now, in front of the current media, which is **kept** and plays again when the splash ends.

- **SplashResume** (or `SplashReturn`, `playevent` type 3): the kept media **continues where it was interrupted** (from the nearest key frame before that point).
- **SplashReset** (`playevent` type 2): the kept media **restarts from its beginning**.
- If the screen was showing only its background, the same applies to the background: SplashResume continues it, SplashReset restarts it.
- If something replaces the splash before it ends, the kept media still comes back afterwards, with the same rule.

### Queue actions

`QueueSameP` and `QueueAlways`:

- **Screen playing:** the media is inserted directly **behind the current one**, without interrupting it. Several queued requests therefore play **newest first** (last in, first out).
- **Screen idle:** plays immediately, like Normal.
- `QueueSameP` still has to pass the priority check (step 7): equal or higher priority queues, lower is refused.
- `QueueAlways` queues even with a lower priority. The queued item keeps its own (lower) priority when it plays.

### SkipSamePrty

Plays like Normal, unless the screen is playing something with the same, non-zero priority, in which case it's refused. Good for frequent events (jackpot calls) that shouldn't cut each other off. At priority 0 it never skips (but it also ignores rest time).

### SetBG

See [the background](#the-background).

### StopPlayer, StopFile

See [stopping](#stopping).

## The background

The background is a file the screen loops whenever its queue is empty.

**Setting it:**

| Method | Effect |
|---|---|
| Trigger action `SetBG`, `playevent …,6,…`, `screens.pup` PlayList column | Checked like any request first: priority 999 stops the screen instead, and a priority **lower** than the current media's is refused (the background slot is left as it was). The rest gate doesn't apply. If it passes, the file becomes the background, with the request's volume, and **starts now**: the current media **and the queue** are dropped. If the file is already what's playing (the current media, or the background loop), it isn't restarted, only its volume changes. |
| `SetBackGround n,1` (`pDMDBackground 1`, `pDMDBackLoopStart`) | The **current media** is copied into the background slot. The media itself plays on to its end as usual (it's still "playing" for priorities, and the queue behind it is kept), and from then on it loops as the background. Needs a current media. **On ForcePop, ForcePopBack and MusicOnly screens the background slot always ends up empty**: there it clears any background, even one set by SetBG. |

**Clearing it:**

| Method | Effect |
|---|---|
| `SetBackGround n,0` | Clears the background slot, however it was set. A background already looping keeps playing until the next stop or replacing request, after which nothing comes back. |
| A new SetBG | Replaces it. |
| A SetBG **image** (png/jpg/jpeg/bmp) | Clears the background video and sets the underlay instead (see [images](#images-in-a-normal-playlist)). |

**When it plays:**

- After any media ends (or is stopped) with nothing queued, the background starts **from its beginning**, except after a SplashResume, where it continues.
- A background whose file name starts with `-` ducks the other screens ([chapter 7](07-volume-ducking-sounds.md#ducking)).

## Stopping

| Request | Stops when |
|---|---|
| `playstop n`, FN 12 | always |
| FN 300 | always, but **without** an end of media ([below](#safe-mode-and-hard-stop)) |
| FN 29 with `FT` 0 or 2 | at the end of the [fade-out](#fading-a-screen-out-fn-29) |
| priority `999` (any request, SetBG included) | a current media is playing (a screen showing only its background is left alone) |
| `StopPlayer` (trigger, `playevent` type 4) | its priority is **0** or **higher** than the current priority |
| `StopFile` (trigger, `playevent` type 5) | the **current** media is exactly that file (queued copies aren't checked). An empty file name compares against, and consumes, the playlist's next file. |

A stop always:

- drops the current media **and the whole queue**,
- counts as an **end of media** ([below](#when-media-ends)),
- restarts the background from its beginning if there is one (or leaves it running if it was already the only thing playing), otherwise leaves the screen empty. A disabled (`Off`) screen is left empty.

### Safe mode and hard stop

| Call | Effect |
|---|---|
| FN 300 | **Hard stop**: if something plays (current media or background), the queue is dropped and the player stops. This is **not** an end of media: nothing restarts (no background), and FN 34, `@`, ducks and splash pages aren't resolved. The background slot is kept, so the next stop or end of media brings it back. |
| FN 24 `EN:1` | **Safe mode** (1.5 engine only): the screen stops like FN 300, followed by an end of media if something was playing, and from then on **every** play request on it is refused, special playlists (PuPFrames, PuPOverlays, PuPShapes) and images included. |
| FN 24 `EN:0` | Ends safe mode. Nothing restarts until the next play. |

On the 2.0 engine FN 24 is ignored. Other `EN` values are ignored.

### Fading a screen out (FN 29)

FN 29 fades the whole screen (picture and sound) out, then ends it:

```vbscript
PuPlayer.SendMSG "{'mt':301,'SN':2,'FN':29,'FT':2,'FI':100}"
```

- Every `FI` milliseconds (default 100; values of 30 or less keep the previous interval), a counter that starts at 255 drops by 10. The screen's opacity becomes counter/255 and the current volume is scaled by the same factor. At 100 ms the fade takes about 2.6 s.
- At the end, `FT` decides: `0` stop, `1` pause (resume with `playresume`), `2` stop and hide, `3` pause and hide. The hide is FN 28's, so it lasts until the screen's next media start (with a background, its next loop pass). After FN 301 on the screen, `1` and `3` stop instead of pausing. Other `FT` values just end the fade.
- Then the screen's opacity goes back to fully opaque (an FN 20 opacity is lost), and its default volume becomes the volume of the media that was faded.
- FN 29 is ignored while a fade (this one, or a duck fade on the screen) is running. `FI` also becomes the step interval of later duck fades on the screen.

> **Rule (one fade-out at a time).** The fade-out counter is shared by every screen. Starting FN 29 on a second screen while another one fades restarts the counter for both and makes both fade faster, and the screen that doesn't finish first may stay faded. Fade one screen at a time.

## When media ends

An **end of media** happens every time something on the screen stops playing a pass of a file:

- the file reaches its end, including **each pass of a looping media or background**,
- its Length runs out, or it fails to decode,
- it's stopped (`playstop`, priority 999, StopPlayer, StopFile, FN 12, a mode change from a drawn mode to `Off` or `MusicOnly`),
- the queue moves on to the next item.

At each end of media:

- labels hidden by FN 34 or by an `@` media ([chapter 10](10-labels.md#hiding-labels-during-a-media)) come back,
- an armed `hidehudplay` returns to the default page and shows pBackground, and an armed `returnplay` returns to the default page ([chapter 10](10-labels.md#pages)),
- if this screen started a duck, the other screens are restored ([chapter 7](07-volume-ducking-sounds.md#ducking)), and a `-NN` file that loops ducks them again on its next pass,
- then, unless it was a loop pass (the looping media simply continues), the next queued item plays (items whose Length ran out while waiting are dropped), or the background starts.

In [jukebox mode](#jukebox-mode-fn-30-and-31), a current media that reaches its end or its Length, and every loop pass, plays the playlist's next file instead.

> **Rule.** Because loop passes count, a `hidehudplay` splash page shown while only a looping background plays returns at the end of the background's current pass.

Media housekeeping runs on its own timer, so all of this also happens on screens that aren't drawn.

## Jukebox mode (FN 30 and 31)

FN 30 with `PM` non-zero turns **jukebox mode** on for the screen. It **can't be turned off** (FN 30 `PM:0` is ignored) until the pack reloads.

- When the current media reaches its end or its Length, or the current media or background finishes a loop pass, the **next file of its playlist** plays instead of the usual queue or background step: an end of media happens, then the file plays as a Normal request at priority 1 and the screen's default volume.
- Stops (`playstop`, priority 999, StopPlayer…) don't advance the jukebox: they end the media as usual.
- FN 31 skips while jukebox mode is on: `PM:1` plays the **previous** file of the playlist, any other positive `PM` the **next** one, now, as a Normal request at priority 1 and the default volume (so a current media above priority 1 refuses it). Without jukebox mode, FN 31 is ignored.
- Each jukebox step switches the playlist's rest time off (on this screen's copy of the playlist), so it never holds back later plays.

## Length, loop and pause from the script

| Call | Applies to |
|---|---|
| `SetLength n, secs` | The media playing **right now** ends `secs` seconds **from now** (0 or less: immediately). On a background, the background ends then and restarts from its beginning. On a static image, a value above 0 sets the image's display time (0 leaves the image up). |
| `SetLoop n, 1/0` | The current media. On a screen that only plays its background, it's ignored: the background keeps looping. |
| `playpause n` / `playresume n` | Pause and resume the current video (and its audio). The queue doesn't move while paused. |
| FN 51 `SP` | Seek what's playing (current media or background) to `SP` milliseconds, from the nearest key frame before it. Ignored if nothing plays, or if `SP` isn't below the file's length (or the file doesn't give one). |
| FN 35 `FT`, `FP` | Jump by `FP` percent of the file's length: `FT:1` forward (at most to 2 s before the end), `FT:2` backward (at least to 2 s from the start). Ignored if nothing plays. |
| FN 41 | [Safe loop](#loop) for what's playing. |

A request's Length (trigger Length column, `playevent` seconds) belongs to that media only, and is counted **from the request**, not from when the media starts. A queued item whose Length runs out while it waits is dropped. A refused request's Length and loop have no effect on what's playing.

## Images in a normal playlist

A `png`, `jpg`, `jpeg` or `bmp` from an ordinary playlist, requested by name or picked as the next file, is handled one of two ways, depending on the screen's **custom image mode**:

| | Custom image mode | Turned on by | Turned off by |
|---|---|---|---|
| 1.5 engine | off by default | FN 7 `CP:1`, FN 36 | FN 7 with any other `CP`, FN 12 |
| 2.0 engine | **on** by default | FN 7 `CP:1`, FN 36 | FN 7 with any other `CP`, FN 12 |

### Without custom image mode: images are media

The image plays through the media engine like a video: priorities, rest, queue, loop and Length all apply, it counts as "playing", and its start and end are ordinary media starts and ends.

- **1.5 engine:** the image is shown for **10 seconds** (paused time doesn't count), then it ends. A shorter Length ends it earlier; a longer one doesn't extend it. Looping (or as a background), it stays up, with an end of media every 10 s.
- **2.0 engine:** the image is a **one-frame** media: it ends right after its frame is shown, and the screen clears about 0.1 s later. As a background it loops that frame.
- On the 2.0 engine a `bmp` is always played this way, even in custom image mode.

### In custom image mode: the static image

The image skips the media engine entirely (no priority, rest or queue).

- **As SetBG:** the image becomes the screen's **underlay** (like a PuPFrames image) and any background **video** is cleared.
- **Otherwise:** what's playing on the screen (current media, queue and background loop) is **stopped without an end of media**, and the image becomes the **static image**. The background slot is kept but doesn't restart until the next stop or end of media. The static image is drawn only while **no video** (current or background) is playing on the screen, and stays until another image replaces it, an accepted video request clears it, or its time runs out: a trigger's Length, or `SetLength`/`playevent` seconds. 0 means forever.
- Either way the image counts as a media start for the screen (shown again if hidden, brought to the front).

> **Rule.** A static image replaces a background video until something restarts the background (a stop, or the end of a later media). Once a background plays again, it covers the static image. Use a PuPOverlays image or an image label to put a picture over a looping background.

A missing image is logged and leaves the screen alone. A stop (`playstop`, FN 12…) doesn't remove the static image: it stays up until another image replaces it, its time runs out, or a video request on the screen is accepted. Playing the image that's already shown doesn't reload it.

`gif` and other formats go through the video decoder and behave like a video. An animated GIF plays like a short video clip.

## Visual continuity

- When one video replaces another, the **last frame of the old one stays on screen** until the new one shows its first frame. There's no black flash.
- When a video ends with nothing to follow (no queue, no background), the screen clears about 0.1 s later.
- If the new file can't be opened, the held frame is dropped.
- On the 2.0 engine, FN 55 (or FN 56, the same with an unused `FH` field) makes a new **current media** video **fade in**: its opacity starts at `FS`/255 and rises by `FS`/255 every drawn frame. `FS` 255 (the default) or 0 disables the fade, and a negative `FS` is ignored. Background (re)starts don't fade. On the 1.5 engine FN 55 and FN 56 are ignored and nothing fades in.

## Masks

A `PuPShapes` image is applied to every video frame on the screen: pixels whose color equals the image's top-left pixel become transparent. It's scaled to the video's frame size, and it stays until a new shape replaces it. Underlay, static image and labels are not masked.
