# 10. Labels: the text and image overlay

Every screen can carry a **label overlay**: named text and image items drawn above the screen's video, grouped into **pages**. This is how PuPDMD packs draw scores, ball counts, splash text and HUD frames.

## The overlay and pBackground

The overlay is created by `PuPlayer.LabelInit screen`, or implicitly by the first `LabelNew` on that screen. It starts with one label already in it:

### pBackground

- Named `pBackground` (any case, `pBACKGROUND` works too).
- Shown on **all pages** (page −1), visible, at the **bottom** of the label order.
- Starts empty. Give it an image caption, or play a `PuPOverlays`/`PuPAlphas` file on the screen, and that image is drawn **stretched over the whole screen**, above the video and below every other label. This is the "HUD frame" of most PuPDMD packs.
- `pDMDSetHUD 0/1` shows and hides it. The `hidehudplay` page special hides it until the current video ends.

You can't create your own label named `pBackground`: the name is taken.

## Fonts

Labels can use:

- every `.ttf` and `.otf` file in the pack's **`FONTS`** folder, loaded with the pack,
- **`Liberation Sans`**, built into VPX (also used by pBackground).

**System fonts are not used.** A pack that names `Impact`, `Arial Black` or any other Windows font without shipping it in `FONTS` gets **no text at all** for those labels, and the log says `Font not found: label=…, font=…`.

A font is found by, in this order (case-insensitive, double spaces collapsed to one):

1. its **family name** as stored in the font file (`Galpon Black`),
2. family + style for non-Regular styles (`Albertus Nova Black`, `Roboto Bold`),
3. its **file name without extension** (`albertusnovablack`).

> **Rule.** When several files share a family name (Regular, Bold, Italic of the same family), the plain family name points to **one of them, unpredictably**. Use the family+style name or the file name to pick a specific one.

> **Rule.** A text label with an **empty font name** never draws text. Image labels don't need a font: `""` is fine for them.

## Creating labels: LabelNew

```vbscript
PuPlayer.LabelNew screen, name, font, size, color, angle, xAlign, yAlign, xPos, yPos, page, visible
```

| Argument | Meaning |
|---|---|
| `screen` | Screen number. The **screen must already exist** (from `screens.pup` or `Init`). Otherwise all label calls on it log `Screen not found` and do nothing. The overlay is created if needed. |
| `name` | Label name, case-insensitive. **A name that already exists is rejected** (logged as `Duplicate label`). The first definition stays. |
| `font` | Font name, see above. |
| `size` | Text height in **percent of the screen's height**, as a full Windows-style cell height (ascent + descent). `10` on a 1080-pixel-high screen is a 108-pixel cell. |
| `color` | BGR integer (`RGB(r,g,b)`). |
| `angle` | Rotation in **tenths of a degree** (`900` = 90°). |
| `xAlign` | 0 left, 1 center, 2 right |
| `yAlign` | 0 top, 1 center, 2 bottom |
| `xPos`, `yPos` | Anchor position in **percent** of the screen's width/height. |

> **Rule (whole numbers only).** `LabelNew`'s `size`, `xPos` and `yPos` are integers: `12.5` becomes 12 or 13 (VBScript rounding). For fractional values, follow up with a `LabelSet` special (`fonth`, `xpos`, `ypos` accept decimals), as `pupCreateLabel` does.
| `page` | Page number. **Negative = shown on every page.** |
| `visible` | 1/0. Also remembered as the label's **default visibility**, which page switches restore ([pages](#pages)). |

### Positioning

Each axis is placed like this (W = screen width, w = label width; the same goes for y with heights):

| Align | Anchor = `xPos` % of W | Label placed with its… |
|---|---|---|
| 0 left/top | `xPos` % | left/top edge on the anchor |
| 1 center | `xPos` % | center on the anchor |
| 2 right/bottom | `xPos` % | right/bottom edge on the anchor |

> **Rule (position 0).** A position of exactly `0` means "the natural spot for this alignment": center alignment → 50 %, right/bottom → 100 %, left/top → 0 %. So `LabelNew …,1,1,0,0,…` is centered on the screen, not at the corner. To really place a centered label's center at 0 %, set the `v2` property ([chapter 11](11-label-properties.md)), as `pupCreateLabel` does.

> **Rule.** Positions are anchors, not distances from an edge. Right alignment with `xPos 95` puts the label's right edge at 95 % of the width.

For text, vertical alignment uses the font's line height times the number of lines, so text is centered on its cell, not on its ink.

Positions and sizes are relative to the **screen's rectangle**. On a CustomPos child, that's the child's box.

## Setting labels: LabelSet

```vbscript
PuPlayer.LabelSet screen, name, caption, visible, special
```

`LabelSet` does these things, in this order:

1. **FN 34 check.** If the screen's labels are hidden by FN 34, the whole call is **ignored**.
2. **Lookup.** An unknown label is logged **once per name** and the call is ignored.
3. **Caption** is applied (rules below).
4. **Visibility** is set to `visible`, but **only if** the caption was `` `u` `` **or** the label is on the **current page**, and **only if** the label isn't animating.
5. **Special** (the JSON in the last argument) is applied: property changes (`mt:2`, [chapter 11](11-label-properties.md)) or an animation (`mt:1`, [chapter 12](12-label-animations.md)).
6. **Dispose.** If the special had `dispose`, the label is deleted.

> **Rule (visibility on other pages).** `LabelSet name, "text", 1, ""` on a label whose page isn't showing changes the text but **not** the visibility. Use the `` `u` `` caption to change the visibility of a label on another page: `LabelSet name, "`u`", 1, ""`.

> **Rule (visibility during animation).** While a label is animating, `LabelSet` can't change its visibility. When the animation ends, the label goes back to the visibility it had when the animation **started**. To hide an animating label, use the `stopani` or `anihide` property.

### Caption rules

| Caption | Effect |
|---|---|
| `` `u` `` (backtick, u, backtick) | Caption **unchanged**. Use it to change only visibility or properties. |
| text | The label shows this text. |
| text with `~` | `~` is a **line break** (or two spaces in pad mode, FN 46). |
| text with `\r` (backslash + r, typed literally) | Also a line break. |
| `TEXT|16744448` | The part before `|` is the text, the number after it sets the label's **color**. An unparsable number gives `16744448`. The color applies even if the text didn't change. |
| `""` (empty) | A text label draws nothing. **On an image label, an empty caption is ignored** and the image stays. |
| a path ending in `.png`, `.jpg`, `.jpeg`, `.bmp` | The label becomes an **image** label. |
| a path ending in `.gif`, `.apng`, `.agif` | The label becomes an **animated image** label. |

- Setting the same text again does nothing (no re-render).
- Text is never wrapped automatically. Only `~` breaks lines, and each line is aligned according to `xAlign`.
- Characters the font doesn't have are drawn as spaces. Accented and other UTF-8 characters work if the font has them.
- Leading and trailing spaces are kept and take up width. Many packs pad captions as `" " & score & " "` so shadows and outlines aren't clipped. Pad mode (FN 46) does this automatically.

### Image captions

An image caption is a path. It's resolved in this order:

1. **Playlist path:** the first path component is taken as a playlist name on this screen and the rest as a file inside it: `"PuPOverlays\frame.png"`, `"dmdmisc\\jewel.png"`, `"Icons/player2/p2.png"`.
2. **Pack path:** otherwise, a path relative to the pack folder.

`\`, `/` and doubled separators all work here, and leading separators are ignored: `"\\Heros\\icon.png"` is the same as `"Heros\\icon.png"`.

> **Rule (missing image).** If the image file doesn't exist, the caption is **cleared** and the label becomes an empty text label. If it was showing an image before, it's also **hidden**. That's what the common "clear.png" idiom relies on: tables "clear" image labels by setting a file that doesn't exist (and logs `Image not found, hiding label`).

> **Rule (images stay images).** Once a label shows an image, setting a plain **text** caption doesn't turn it back into text. The text is stored, but the image stays on screen. Only another image caption, or a missing-image caption (which hides it), changes it. Use separate labels for text and images.

Image labels are drawn at the image's **native pixel size** until `width`/`height` are set ([chapter 11](11-label-properties.md)), except **pBackground**, which is always stretched to the screen. When the same image path is set again, nothing reloads. A new path restarts its animation.

### Animated images (GIF, APNG)

Animated images play their frames using the file's own frame delays, looping, until told otherwise. The properties `animate`, `anigif`, `anistart`, `gifstart`, `gifend`, `gifloop`, `aniendhide`, `anidispose` control them ([chapter 11](11-label-properties.md#animated-images)).

## Pages

A label is drawn when **all** of these hold:

- it's visible, **or** it's animating,
- its page is **negative**, **or** equal to the screen's **current page**,
- its caption isn't empty,
- the overlay is on (FN 3) and not hidden by FN 34.

The current page and the default page both start at **0**.

### LabelShowPage

```vbscript
PuPlayer.LabelShowPage screen, page, duration, special
```

> **Rule (seconds vs milliseconds).** `duration` **below 501 is seconds**, **501 and above is milliseconds**. `3` = 3 s, `500` = 500 **seconds**, `501` = 0.501 s, `2500` = 2.5 s.

What happens:

1. If the screen's labels are hidden by FN 34, the call is **ignored**.
2. **Special `hidehudplay`** (any case): pBackground is hidden now, and a return is armed. When the screen's current media ends, go back to the default page and show pBackground again.
   **Special `returnplay`** (any case): a return is armed. When the current media ends, go back to the default page.
   Any other special disarms a pending return.
3. **Special `forceshow` / `forcehide`** (exactly lowercase): no page change. Every label on `page` is set to its default visibility (`forceshow`) or hidden (`forcehide`). Done.
4. **Same page, nothing armed:** if `page` is already the current page, all label animations on the screen are stopped (labels restored), and a running page timer is **extended** by `duration`. Nothing else changes. Done.
5. **Page switch:**
   - all label animations on the screen are stopped,
   - special **`clear`**: every **text** label on the new page gets an empty caption (image labels keep their image),
   - if `page` is **greater than 1**, every label on the new page whose `dodefault` is on (the default) goes back to its **default visibility** (the `visible` it was created with). Pages 0 and 1 never do this,
   - `duration` > 0: after that time the screen returns to the **default page**,
   - `duration` ≤ 0: the page becomes the new **default page** (if it's 0 or more).

> **Rule (splash pages need a video).** `hidehudplay` and `returnplay` return at the screen's next **end of media**: the current video's end, a stop, a queue advance, or a loop pass of a looping media or background ([chapter 6](06-playback-engine.md#when-media-ends)). If nothing plays at all, only the duration timer brings the page back, and the PuPDMD helpers pass `500`, which is **500 seconds**. A timer return (or any other `LabelShowPage`/`pDMDSetPage` while the return is armed) only switches the page: after `hidehudplay`, **pBackground stays hidden** until you show it (`pDMDSetHUD 1`). Only a real end of media restores it.

> **Rule (default visibility).** Labels created hidden (`visible` 0) on pages 2 and above are **re-hidden on every switch** to their page, even if you showed them before. Create labels visible if they should appear whenever their page shows, or set `dodefault:0` on them.

The page timer runs whether or not the screen is drawn.

## Label order

Labels are drawn in creation order (pBackground first), then **every animating label on top**, whatever its position. The `ztop` and `zback` properties change the order ([chapter 11](11-label-properties.md#order)).

## Rendering details

- **Text** is rendered with anti-aliasing, re-rendered whenever its text, font, size, color, style, shadow or the screen's size changes.
- **Bold is not rendered**: the `bold` property is accepted but has no visible effect. Italic and underline work.
- **Shadow** (drop shadow) and **outline** are drawn around the text in the shadow color ([chapter 11](11-label-properties.md#shadow-and-outline)).
- **Rotation** (`angle`, `rotate`, wiggle) turns the label around its center. The rotated label is still clipped to the screen.
- **Zoom** scales the label around its center.
- **Alpha:** label alpha × screen alpha (FN 20) × overlay alpha (FN 22).
- All labels are clipped to the screen's rectangle.
