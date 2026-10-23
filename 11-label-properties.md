# 11. Label special properties (`mt:2`)

The last argument of `LabelSet` is a JSON object. With `"mt":2` it changes label properties:

```vbscript
PuPlayer.LabelSet pDMD, "Score", "`u`", 1, "{'mt':2,'color':255,'xpos':50,'ypos':80}"
```

- Single quotes are fine, and so are double quotes (doubled inside VBScript strings).
- `mt` 0, or no `mt`, does nothing. `mt` 1 is an animation ([chapter 12](12-label-animations.md)). Any other `mt` is logged.
- The special is applied **after** the caption and visibility of the same call. A `width` for an image works in the same call that sets the image caption.
- An empty special (`""`) does nothing.
- Unknown keys are logged (`<key> not implemented`) and skipped. The other keys still apply.

> **Rule (key order).** The keys of one special are applied in a **fixed order**, not in the order you write them: `size`, `fonth`, `xpos`, `ypos`, `width`, `height`, `autow`, `autoh`, `zoom`, `xoffset`, `yoffset`, `anistart`, `anigif`, `animate`, `fname`, `color`, `xalign`, `yalign`, `pagenum`, `quality`, `rotate`, `alpha`, `xoffpix`, `yoffpix`, `qual`, `bold`, `italic`, `underline`, `normal`, `shadowcolor`, `shadowtype`, `shadowstate`, `outline`, `gradcolor`, `gradstate`, `grayscale`, `filter`, `dodefault`, `maxlen`, `transcolor`, `gifstart`, `gifend`, `gifloop`, `anidispose`, `anihide`, `aniendhide`, `rescale`, `ztop`, `zback`, `forceshow`, `forcehide`, `stopani`, `showstats`, `v2`, `dispose`. So in one special, `shadowstate` is applied before `outline`, `fonth` after `size`, and `gifstart` before `gifend`.

## Position and size

| Key | Value | Effect |
|---|---|---|
| `xpos`, `ypos` | percent (decimal) | New anchor position ([chapter 10](10-labels.md#positioning)). |
| `xalign` | 0, 1, 2 | Horizontal alignment: left, center, right. Also how multi-line text is justified. |
| `yalign` | 0, 1, 2 | Vertical alignment: top, center, bottom. |
| `v2` | any | Position **0 means 0 %** for every alignment, instead of "center/right/bottom" ([chapter 10](10-labels.md#positioning)). Can't be switched off again. |
| `fonth` | percent | Text height as a **cell height** in percent of the screen height, the same unit as `LabelNew`'s `size`. |
| `size` | percent (≥ 1) | Text height as a **point size**, which comes out larger than the same number in `fonth`/`LabelNew`: about 1.33 ÷ (the font's em-to-cell ratio), typically **1.4 to 1.7 times** larger. |
| `width`, `height` | percent of screen width / height | **Image labels only**: the drawn size. Until set, an image is drawn at its native pixel size. Ignored while the label holds text, so set the image caption first (the same call works). Ignored for pBackground (always stretched). |
| `autow`, `autoh` | percent of screen width / height | **Auto-fit text**: if the rendered text is wider than `autow` % or taller than `autoh` %, the font shrinks in 2-pixel steps until it fits (minimum 4 px). 0 = no limit. |
| `zoom` | percent | Scale the label around its center (`120` = 120 %). Applies to text and images. |
| `rotate` | tenths of a degree | Rotation (`900` = 90°), text and images. Use center alignment for images so they rotate in place. |
| `pagenum` | page | Move the label to another page. |

## Color and look

| Key | Value | Effect |
|---|---|---|
| `color` | BGR | Text color. |
| `alpha` | 0–255 | Label opacity (`255` opaque, `0` invisible). |
| `gradcolor` | BGR | End color of a vertical gradient (top = `color`, bottom = `gradcolor`). |
| `gradstate` | 0–100 | Gradient: **0 = off**. Otherwise the percentage of the text height (from the top) kept in the start color before the gradient begins. `1` = gradient over the whole height, `50` = top half solid. |
| `grayscale` | 1/0 | Shortcut for `filter` 3 (1) or no filter (0). |
| `filter` | 0–6 | Pixel filter on the label (text or image): `0` none, `1` invert RGB, `2` invert RGB and alpha, `3` grayscale, `4` invert alpha, `5` clear (invisible), `6` blur (box blur, radius 5 px). |
| `bold` | 1/0 | Accepted, **not rendered**. |
| `italic` | 1/0 | Italic text. |
| `underline` | 1/0 | Underlined text. |
| `normal` | any | Clears bold, italic and underline. |
| `fname` | font name | Change the font ([chapter 10](10-labels.md#fonts)). A font that isn't found leaves the label **without** a font (no text) and is logged. |

### Shadow and outline

| Key | Value | Effect |
|---|---|---|
| `shadowstate` | 0 / non-zero | 0 = no shadow, anything else = **drop shadow**. |
| `shadowtype` | 0, 1, 2 | 0 none, 1 drop shadow, 2 **outline** (border). |
| `outline` | 0/1/… | With a drop shadow already on: `1` switches to outline, `0` keeps the drop shadow. Without one: `1` → outline, other values → that shadow type (0 → none). |
| `shadowcolor` | BGR | Shadow / outline color. **Default grey (`8421504`).** |
| `xoffset`, `yoffset` | percent of the font height | Shadow offset, or outline thickness. |
| `xoffpix`, `yoffpix` | pixels | Same, in pixels. **Default 3 × 3 px.** |

- No shadow is drawn until a shadow type is set. The offsets alone do nothing.
- **Drop shadow:** a copy of the text in the shadow color, offset by (x, y). Negative offsets go up/left. The label grows by the offset.
- **Outline:** a border around the glyphs, about `max(|x|, |y|) − 2` pixels thick (at least 1). The label is shifted left by twice the x offset to keep the text roughly in place. An outline with both offsets 0 draws no outline.

```vbscript
' Drop shadow, black, 4 % of the font height
PuPlayer.LabelSet pDMD, "Score", "`u`", 1, "{'mt':2,'shadowcolor':0,'shadowtype':1,'xoffset':4,'yoffset':4}"
' White outline
PuPlayer.LabelSet pDMD, "Ball", "`u`", 1, "{'mt':2,'shadowcolor':16777215,'shadowtype':2,'xoffset':3,'yoffset':3}"
```

## Visibility and lifetime

| Key | Value | Effect |
|---|---|---|
| `dodefault` | 1/0 | Whether page switches (to pages > 1) restore the label's default visibility. Default 1. |
| `maxlen` | characters | Truncates the label's **current** text caption to this length. Later captions are not truncated. Send it after the caption (same call works). |
| `stopani` | any | Stops the label's animation (zoom/flash/move…) and restores its state. Animated GIF frames keep playing. |
| `anihide` | any | Stops the label's animation **and hides** it. |
| `forceshow` / `forcehide` | any | Show / hide the label (its visibility flag), whatever page it's on. |
| `dispose` | any | Stops animations, hides the label, and **deletes** it after this call. It no longer exists: later `LabelSet`s on it log `Invalid label`. Re-create it with `LabelNew` if needed. |

## Order

| Key | Value | Effect |
|---|---|---|
| `ztop` | any | Move the label to the **top** of the label order. |
| `zback` | `-1` | Move the label to the **very bottom**, below pBackground. |
| `zback` | any other value | Move the label to **just above pBackground**, below all other labels. |

The PuPDMD helper `pDMDLabelSendToBack` sends `zback:1`, so "send to back" really means "back, but still above the HUD frame".

Animating labels are always drawn above non-animating ones, whatever the order.

## Animated images

These apply to GIF / APNG image labels.

| Key | Value | Effect |
|---|---|---|
| `animate` | ms per frame, or 0 | `> 0`: play the frames at this fixed interval (instead of the file's own delays), restart from the first frame, and make the label **visible**. `0` or negative: **stop** at the current frame. |
| `anistart` | ms per frame, or 0 | Like `animate`, without forcing visibility. `0` or negative **stops**. |
| `anigif` | speed 0–100 | Start animating at a speed-derived interval: `100` or more → 40 ms per frame. Lower values → `30 + (100 − speed) × 3` ms, so `0` → 330 ms, `50` → 180 ms. |
| `gifstart` | frame index | First frame of the played range. `-1` resets to the full range. Restarts the animation. |
| `gifend` | frame index | Last frame of the played range. **`gifstart` = `gifend` shows that single frame** (an image list). |
| `gifloop` | 1/0 | 1 loop forever (default), 0 play once and stop on the last frame. |
| `aniendhide` | 1/0 | Hide the label when a play-once animation finishes. |
| `anidispose` | 1/0 | When a play-once animation finishes, release the image: the label becomes empty (but still exists). |

Without any of these, an animated image plays with its own frame delays, looping.

```vbscript
' Show frame 5 of an image strip
PuPlayer.LabelSet pDMD, "Icons", "`u`", 1, "{'mt':2,'gifstart':5,'gifend':5}"
' Play frames 3..9 once at 30 ms per frame, then hide
PuPlayer.LabelSet pDMD, "Boom", "`u`", 1, "{'mt':2,'gifstart':3,'gifend':9,'gifloop':0}"
PuPlayer.LabelSet pDMD, "Boom", "`u`", 1, "{'mt':2,'animate':30,'aniendhide':1}"
```

## Keys that are accepted but not supported

`qual`, `quality`, `rescale`, `transcolor`, `showstats`. These are logged and ignored.
