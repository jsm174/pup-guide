# 6. The playback engine

Every screen runs its own media engine. This chapter gives its exact rules: what happens to a play request, how priorities, the queue, splashes and the background interact, and what happens when media ends.

## A screen's state

| Part | What it is |
|---|---|
| **Current media** | The media playing now (the head of the queue), with its priority, volume, length and loop flag. |
| **Queue** | Media waiting behind the current one. |
| **Background** | A file looped whenever the queue is empty. |
| **Underlay** | A `PuPFrames` image under the video. |
| **Static image** | A `png/jpg/jpeg/bmp` played from a normal playlist, shown when no video is playing. |
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

A request goes through these steps, in this order. "Refused" means nothing changes on the screen.

**At the screen:**

1. **Special playlists.** If the playlist is `PuPFrames`, `PuPOverlays`, `PuPAlphas` or `PuPShapes`, the file is applied as an underlay, overlay or mask ([chapter 4](04-playlists.md#special-playlists)). Done. No priority, queue or volume involved.
2. **Images.** If the file (the requested one, or the playlist's next file) ends in `png`, `jpg`, `jpeg` or `bmp`, see [images](#images-in-a-normal-playlist). Done. No priority involved.
3. **Off screen.** If the screen is `Off`, refused.
4. **Reordering.** A `ForcePop` screen moves to the front of its draw order, a `ForcePopBack` screen to the back. This happens **even if the request is refused later**.
5. If the media engine accepts the request below, the screen's static image is cleared.
6. **Volume** is resolved: −1 becomes the playlist volume, then the value is limited to 1–200.

**In the media engine:**

7. **Priority 999** (any action except SetBG): if a current media plays, stop the screen ([stopping](#stopping)). Either way, play nothing. Done.
8. **Lower priority**: if the screen is playing and the request's priority is **lower** than the current priority, refused, unless the action is `QueueAlways`.
9. **SkipSamePrty**: if the screen is playing, the priority is above 0 and equal to the current priority, refused.
10. **Rest gate**: see [below](#the-rest-gate).
11. **File lookup**: an empty file name takes the playlist's next file (advancing the rotation). A missing file is logged and refused.
12. **SetBG** is handled ([the background](#the-background)). Done.
13. **Same file**: if the current media is the same file, it is **not restarted**. Only its volume is updated. Done.
14. **Background already showing it**: if the screen is idle and its background loop is playing this very file, refused (the playlist rest timer is still restarted).
15. **Queue actions** while playing: inserted behind the current media ([queue actions](#queue-actions)). Done.
16. **Splash actions**: played in front of the current media, which is kept ([splash](#splashreset-and-splashresume)). Done.
17. **Everything else**: the current media is discarded and the new one plays now.

Each accepted request (from step 11 on) also restarts the playlist's rest timer.

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
- `QueueSameP` still has to pass the priority check (step 8): equal or higher priority queues, lower is refused.
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
| Trigger action `SetBG`, `playevent …,6,…`, `screens.pup` PlayList column | The file becomes the background, with the request's volume. If the screen is idle, it starts now (if it's already the playing background, only the volume changes). If the screen is playing, it starts when the current media ends. **Priority, 999 and rest don't apply.** |
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
| priority `999` (any request except SetBG) | a current media is playing (a screen showing only its background is left alone) |
| `StopPlayer` (trigger, `playevent` type 4) | its priority is **0** or **higher** than the current priority |
| `StopFile` (trigger, `playevent` type 5) | the **current** media is exactly that file (queued copies aren't checked). An empty file name compares against, and consumes, the playlist's next file. |

A stop always:

- drops the current media **and the whole queue**,
- counts as an **end of media** ([below](#when-media-ends)),
- restarts the background from its beginning if there is one (or leaves it running if it was already the only thing playing), otherwise leaves the screen empty. A disabled (`Off`) screen is left empty.

## When media ends

An **end of media** happens every time something on the screen stops playing a pass of a file:

- the file reaches its end, including **each pass of a looping media or background**,
- its Length runs out, or it fails to decode,
- it's stopped (`playstop`, priority 999, StopPlayer, StopFile, FN 12, a mode change from a drawn mode to `Off` or `MusicOnly`),
- the queue moves on to the next item.

At each end of media:

- labels hidden by FN 34 come back,
- an armed `hidehudplay` returns to the default page and shows pBackground, and an armed `returnplay` returns to the default page ([chapter 10](10-labels.md#pages)),
- if this screen started a duck, the other screens are restored ([chapter 7](07-volume-ducking-sounds.md#ducking)), and a `-NN` file that loops ducks them again on its next pass,
- then, unless it was a loop pass (the looping media simply continues), the next queued item plays (items whose Length ran out while waiting are dropped), or the background starts.

> **Rule.** Because loop passes count, a `hidehudplay` splash page shown while only a looping background plays returns at the end of the background's current pass.

Media housekeeping runs on its own timer, so all of this also happens on screens that aren't drawn.

## Length, loop and pause from the script

| Call | Applies to |
|---|---|
| `SetLength n, secs` | The media playing **right now** ends `secs` seconds **from now** (0 or less: immediately). On a background, the background ends then and restarts from its beginning. On a static image, a value above 0 sets the image's display time (0 leaves the image up). |
| `SetLoop n, 1/0` | The media playing right now. |
| `playpause n` / `playresume n` | Pause and resume the current video (and its audio). The queue doesn't move while paused. |

A request's Length (trigger Length column, `playevent` seconds) belongs to that media only, and is counted **from the request**, not from when the media starts. A queued item whose Length runs out while it waits is dropped. A refused request's Length and loop have no effect on what's playing.

## Images in a normal playlist

A `png`, `jpg`, `jpeg` or `bmp` from an ordinary playlist, requested by name or picked as the next file, skips the media engine entirely (no priority, rest or queue).

- **As SetBG:** the image becomes the screen's **underlay** (like a PuPFrames image) and any background **video** is cleared.
- **Otherwise:** the screen is **stopped** (current media and queue dropped, background restarted if any), and the image becomes the **static image**. It's drawn only while **no video** (current or background) is playing on the screen, and stays until another image replaces it, an accepted video request clears it, or its time runs out: a trigger's Length, or `SetLength`/`playevent` seconds. 0 means forever.

> **Rule.** On a screen with a background video, a static image never shows: the stop that precedes it restarts the background, which covers it. Use a PuPOverlays image or an image label to put a picture over a looping background.

A missing image is logged and leaves the screen alone. A stop (`playstop`, FN 12…) doesn't remove the static image: it stays up until another image replaces it, its time runs out, or a video request on the screen is accepted. Playing the image that's already shown doesn't reload it.

`gif` and other formats go through the video decoder and behave like a video. An animated GIF plays like a short video clip.

## Visual continuity

- When one video replaces another, the **last frame of the old one stays on screen** until the new one shows its first frame. There's no black flash.
- When a video ends with nothing to follow (no queue, no background), the screen clears about 0.1 s later.
- If the new file can't be opened, the held frame is dropped.
- With FN 55 a new **current media** video **fades in**: its opacity starts at `FS`/255 and rises by `FS`/255 every drawn frame. `FS` 255 (the default) disables the fade. Background (re)starts don't fade. `FS` 0 or negative also disables the fade.

## Masks

A `PuPShapes` image is applied to every video frame on the screen: pixels whose color equals the image's top-left pixel become transparent. It's scaled to the video's frame size, and it stays until a new shape replaces it. Underlay, static image and labels are not masked.
