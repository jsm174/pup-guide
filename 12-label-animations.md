# 12. Label animations (`mt:1`)

A `LabelSet` special with `"mt":1` starts an animation on the label:

```vbscript
PuPlayer.LabelSet pDMD, "Splash", "JACKPOT", 1, "{'mt':1,'at':1,'fq':150,'len':3000,'fc':255}"
```

`at` selects the animation type. All times are **milliseconds of real time**.

## Rules common to every animation

- **One at a time.** A label runs one animation. A new one replaces the running one, which is stopped and restored first. With `ad` non-zero (parallel animations, not supported), the running one is replaced **without** being restored, and a warning is logged. An interrupted filter flash then leaves its filter on the label, and the new animation captures the half-finished state.
- **State capture.** When an animation starts, the label's **visibility, alpha, zoom, color and filter** are captured. When it ends or is stopped, they're **restored**. An animation never leaves those changed, with two exceptions: the number count-up (type 4) and the overlay fade (type 7).
- **The label's own alpha and zoom are replaced while animating.** The animation draws at its own alpha (opaque unless it fades) and its own zoom (100 % unless it zooms). A label set to `alpha 128` is drawn fully opaque while it flashes or moves, and a type-5 fade from 255 ignores the label's alpha.
- **The visible argument decides the end state.** `LabelSet` applies `visible` *before* it starts the animation (if the label is on the current page), so the `visible` you pass is the visibility the label **returns to** afterwards. That's why "scroll" and "fade out" helpers pass 0 and "fade in" helpers pass 1. **Exception:** if the label is *already* animating, the visible argument is skipped, so the new animation inherits the old one's starting visibility. A `pDMDLabelFadeIn` sent during a `pDMDLabelFadeOut` ends **hidden**. Stop the first one (`stopani`) before starting the second.
- **Animations only take effect while drawn.** The clock is real time and keeps running, but an animation's effects, and its end (with the restore), are only applied while its label is drawn: on the current page (or a negative page), with a non-empty caption, overlay on. An animation on a page that isn't showing ends the next time its label is drawn (or when it's cleared).
- **Hidden labels animate.** An animating label is drawn even if it's hidden. The animation decides when it shows.
- **On top.** An animating label is drawn above all non-animating labels of the screen.
- **Stopping.** An animation ends after `len`, or earlier on: a page switch, a `LabelShowPage` of the current page, the `stopani`, `anihide` or `dispose` properties, or a new animation on the label.
- **Invalid types.** An unknown `at` is logged and nothing animates. Types 1, 2 and 9 without their required keys are logged as invalid.

### Common keys

| Key | Default | Meaning |
|---|---|---|
| `at` | | Animation type, 1–10 (table below). |
| `len` | per type | Total duration in ms. |
| `mlen` | = `len` | Duration of the motion, zoom, fade or count part. After it, the end state is held until `len`. The fallback copies the `len` **you sent**: if you leave `len` out and rely on a type's default length, `mlen` is 0 and a zoom (type 3) or count-up (type 4) **jumps** to its end value immediately. Fades fall back to the full length. |
| `tt` | 0 | Easing curve of the motion part (below). |
| `fc` | label color | Text color **during** the animation. |
| `af` | 0 | If above 50: **fade out** during the last `af` ms. Types 1, 2, 3, 4 and 10 only. |
| `xp`, `yp` | | Types 1, 2 and 10: draw the animated label with its top-left corner at this **pixel** position in the screen (whole pixels, decimals dropped), instead of its normal position. |
| `ad` | 0 | Parallel animations. Not supported (replaces anyway). |

### Easing curves (`tt`)

| `tt` | Curve |
|---|---|
| 0 | linear |
| 1 | quadratic in |
| 2 | quadratic out |
| 3 | quadratic in-out |
| 4 | cubic in |
| 5 | cubic out |
| 6 | cubic in-out |
| 7 | quartic in |
| 8 | quartic out |
| 9 | quartic in-out |
| 10 | quintic in |
| other | linear |

## Animation types

### `at:1`: flash

```
{'mt':1,'at':1,'fq':150,'len':3000,'fc':255}
```

| Key | Default | |
|---|---|---|
| `len` | **required** | |
| `fq` | **required, > 0** | Toggle interval in ms. |

Starts **hidden**, then toggles visible/hidden every `fq` ms.

### `at:2`: move

```
{'mt':1,'at':2,'len':3000,'xps':1,'xpe':-1,'tt':0,'fc':255}           scroll right to left
{'mt':1,'at':2,'len':1000,'xps':5,'xpe':35,'yps':20,'ype':78,'tt':6,'af':700}
```

| Key | Default | |
|---|---|---|
| `len` | **required** | |
| `xps`, `xpe` | 0 | Start / end x. |
| `yps`, `ype` | 0 | Start / end y. |
| `mColor` | `fc` | Text color while moving. `fc` applies once the motion is done. |

Position values:

| Value | Meaning |
|---|---|
| `0` | the label's normal position (no movement on this axis if both are 0) |
| `1` | just **off-screen** right / bottom |
| `-1` | just **off-screen** left / top |
| `2` (end only) | the label's normal position (or `xp`/`yp` if given) |
| anything else | **percent** of the screen width/height, where the label's **left/top edge** goes. This is not the alignment anchor. |

> **Rule.** `1`, `-1`, `0` and `2` are special values, so you can't move to exactly 1 % or 2 %. Use `1.01` or `2.01`.

The motion runs over `mlen` with easing `tt`, then the label holds its end position until `len`, and then snaps back to its normal position (and the visibility you passed).

### `at:3`: zoom

```
{'mt':1,'at':3,'hstart':0,'hend':100,'len':3000,'mlen':1200,'tt':0,'fc':65280,'ns':1}
```

| Key | Default | |
|---|---|---|
| `hstart`, `hend` | 100, 200 | Start / end size. |
| `ns` | absent | If present: sizes are **percent of the label's own size** (100 = normal). |
| `len` | 1000 | |

Without `ns`, for a **text** label the sizes are text heights in percent of the **screen** height, like the label's font size (with a 10 % font, `hstart:10` is normal size and `hend:20` is double). For an **image** label they are heights in percent of the screen height.

Zooms from `hstart` to `hend` over `mlen` with easing `tt`, holds until `len`, then returns to normal.

### `at:4`: pulse (and number count-up)

```
{'mt':1,'at':4,'hstart':80,'hend':120,'len':3000,'pspeed':0,'fc':255}
{'mt':1,'at':4,'hstart':80,'hend':120,'len':3000,'pspeed':0,'numstart':1000,'numend':2000,'numformat':1}
```

| Key | Default | |
|---|---|---|
| `hstart`, `hend` | 80, 120 | Size range, in percent of the label's normal size. |
| `pspeed` | 0 | Speed: pixels of height per 30 ms. 0 = 3 % of the label's height per 30 ms. |
| `len` | 3000 | |
| `numstart`, `numend` | | If `numend` is present, the caption **counts** from `numstart` to `numend` over `mlen` with easing `tt`. |
| `numformat` | 0 | 1 = thousands separators, padded: `" 1,234 "` (and at least 2 characters, so `" 5 "` becomes `"  5 "`). |

The size bounces between `hstart` and `hend` (triangle wave) until `len`. The counted number **stays** as the caption afterwards.

### `at:5`: fade

```
{'mt':1,'at':5,'astart':255,'aend':0,'len':3000}        fade out
{'mt':1,'at':5,'astart':0,'aend':255,'len':3000}        fade in
```

| Key | Default | |
|---|---|---|
| `astart`, `aend` | 255, 0 | Alpha 0–255. |
| `len` | 1000 | |

Linear over `mlen`, then held until `len`. Afterwards the label's **own alpha** returns, so a fade-out is only an "out" because the helper also passes `visible` 0.

### `at:6`: fade pulse

```
{'mt':1,'at':6,'astart':70,'aend':255,'len':3000,'pspeed':40,'fc':255}
```

| Key | Default | |
|---|---|---|
| `astart`, `aend` | 70, 255 | Alpha range. |
| `pspeed` | 1 | Alpha units per 30 ms (minimum 1). |
| `len` | 3000 | |

Alpha bounces between `astart` and `aend` until `len`.

### `at:7`: overlay fade

```
{'mt':1,'at':7,'astart':255,'aend':0,'len':3000}
```

Like type 5, but fades the **whole label overlay** of the screen (the FN 22 multiplier), not just this label. Defaults 255 → 0, `len` 1000.

> **Rule.** The overlay alpha **stays** at `aend` when the animation ends. After a screen fade-out every label on the screen stays invisible until a fade-in (`astart 0`, `aend 255`) or FN 22 `AM:0`. It's the only animation whose effect persists.

> **Rule.** An overlay fade that's interrupted (page switch, `stopani`, a new animation on the label) leaves the overlay alpha at its **mid-fade** value. And the fade only acts while its label is drawn: on a label with an empty caption it never changes the overlay at all, and on another page it does nothing until that page shows. Use a visible label on a negative page (such as pBackground with an image) to carry screen fades.

### `at:8`: wiggle

```
{'mt':1,'at':8,'rstart':-45,'rend':45,'len':3000,'rspeed':5,'fc':255}
```

| Key | Default | |
|---|---|---|
| `rstart`, `rend` | −45, 45 | Rotation range in **whole degrees** (not tenths, unlike `angle`/`rotate`). |
| `rspeed` | 1 | Degrees per 30 ms (0 → 1). |
| `len` | 3000 | |

The rotation bounces between `rstart` and `rend` until `len`, then the label returns to its normal angle. The wiggle replaces the label's own angle while it runs.

### `at:9`: filter flash

```
{'mt':1,'at':9,'fq':150,'len':3000,'fm':1}
```

| Key | Default | |
|---|---|---|
| `len` | **required** | |
| `fq` | **required, > 0** | Toggle interval in ms. |
| `fm` | 0 | Filter number ([chapter 11](11-label-properties.md#color-and-look)). |

The filter toggles off/on every `fq` ms, starting **off**. The label's own filter is restored at the end.

### `at:10`: delayed show ("clone")

```
{'mt':1,'at':10,'len':3000,'dl':1000,'xp':200,'yp':100,'fc':65535}
```

| Key | Default | |
|---|---|---|
| `dl` | 0 | Delay in ms before showing. |
| `len` | 0 | |

The label stays hidden for `dl` ms, then shows until `len`. With `xp`/`yp` it's drawn at that pixel position. The PuPDMD "clone" helpers pass `visible` 0, so the label disappears again afterwards.
