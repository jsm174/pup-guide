# 14. Authored DMD overlays (`displays.pup`)

A pack can draw the game's **live DMD** (or another display of the controller) on top of its PUP screens. A typical use is a PuP pack whose backglass video leaves a "window" for the real DMD. This is a VPX feature, stored in the pack as `displays.pup`, and it can be edited in game.

## How it renders

- Each entry draws one display into one window: **Backglass**, **ScoreView** or **Topper**.
- Overlays are drawn **after** (on top of) every PUP screen of that window.
- They're drawn even when the window has no usable PUP screen. In that case PUP claims the window just for the overlays.

## File format

`displays.pup` is a JSON file in the pack folder (not CSV like the other `.pup` files):

```json
{
    "Displays": [
        {
            "Target": "Backglass",
            "Source": "",
            "X": 520, "Y": 610, "W": 880, "H": 220,
            "Style": "Neon Plasma",
            "Dots": [128, 32],
            "Tint": [1.0, 0.55, 0.2],
            "Glass": {
                "Image": "PUPDisplays/dmdglass.png",
                "Area": [0, 0, 0, 0],
                "Tint": [1, 1, 1],
                "Roughness": 0.3,
                "Ambient": [1, 1, 1],
                "Pad": [0, 0, 0, 0]
            }
        }
    ]
}
```

A bare top-level array (without `"Displays"`) is also accepted. Invalid JSON is logged and loads no overlays.

| Field | Default | Meaning |
|---|---|---|
| `Target` | `Backglass` | `Backglass`, `ScoreView` or `Topper` (case-insensitive). Anything else is Backglass. |
| `Source` | `""` | The display to show, as a VPX resource URI. Empty = the game's default DMD. |
| `X`, `Y`, `W`, `H` | 0 | Bounds in **window pixels**, origin at the top left. |
| `Style` | auto | Render style: `Neon Plasma`, `Red LED`, `Green LED`, `Yellow LED`, `Generic Plasma`, `Generic LED`, `LCD`, `CRT`. Omitted = derived from the hardware. |
| `Dots` | unset | Expected source size in dots `[w, h]`. |
| `Tint` | `[1,1,1]` | Emitter color multiplier. |
| `Glass.Image` | none | Glass image, path relative to the pack, normally in `PUPDisplays/`. |
| `Glass.Area` | whole image | Part of the glass image used `[x, y, w, h]`, in image pixels. |
| `Glass.Tint` | `[1,1,1]` | Glass color. |
| `Glass.Roughness` | 0 | Glass blur/roughness. |
| `Glass.Ambient` | `[1,1,1]` | Room light reflected by the glass. |
| `Glass.Pad` | `[0,0,0,0]` | Padding of the emitter inside the glass bounds (left, top, right, bottom). |

When VPX saves the file, `Target`, `Source`, `X`, `Y`, `W` and `H` are always written. The other fields are written only when they differ from their defaults.

> **Rule.** `Style` names are **case-sensitive** and must be spelled exactly as listed. Anything else silently means auto.

Because the bounds are window pixels, an overlay authored for one window size is in the wrong place at another size. Author it on the setup it's meant for, or re-adjust it in game.

## Editing in game

With a pack loaded, VPX's in-game settings UI has a sheet **Modify "displays.pup"** listing the overlays. You can add and remove displays, change target, source, bounds, style, tint and glass, pick a glass image, and import a Nixie layout.

Committing the edit rewrites `displays.pup` and **copies any newly picked glass image** into the pack's `PUPDisplays` folder (renamed `name_1.png`, `name_2.png`… if a different file with that name already exists). Cancelling leaves the pack untouched.
